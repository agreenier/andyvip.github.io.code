---
date: 2016-11-24 01:38
title: Property代码解读
tags: [android]
categories: [android,property]
---

#prop_area的数据结构
```C++
struct prop_area {
    uint32_t bytes_used;
    atomic_uint_least32_t serial;
    uint32_t magic;
    uint32_t version;
    uint32_t reserved[28];
    char data[0];

    prop_area(const uint32_t magic, const uint32_t version) :
        magic(magic), version(version) {
        atomic_init(&serial, 0);
        memset(reserved, 0, sizeof(reserved));
        // Allocate enough space for the root node.
        bytes_used = sizeof(prop_bt);
    }
}

/*
 * Properties are stored in a hybrid trie/binary tree structure.
 * Each property's name is delimited at '.' characters, and the tokens are put
 * into a trie structure.  Siblings at each level of the trie are stored in a
 * binary tree.  For instance, "ro.secure"="1" could be stored as follows:
 *
 * +-----+   children    +----+   children    +--------+
 * |     |-------------->| ro |-------------->| secure |
 * +-----+               +----+               +--------+
 *                       /    \                /   |
 *                 left /      \ right   left /    |  prop   +===========+
 *                     v        v            v     +-------->| ro.secure |
 *                  +-----+   +-----+     +-----+            +-----------+
 *                  | net |   | sys |     | com |            |     1     |
 *                  +-----+   +-----+     +-----+            +===========+
 */
 //树节点的数据结构
struct prop_bt {
    uint8_t namelen;
    uint8_t reserved[3];

    // The property trie is updated only by the init process (single threaded) which provides
    // property service. And it can be read by multiple threads at the same time.
    // As the property trie is not protected by locks, we use atomic_uint_least32_t types for the
    // left, right, children "pointers" in the trie node. To make sure readers who see the
    // change of "pointers" can also notice the change of prop_bt structure contents pointed by
    // the "pointers", we always use release-consume ordering pair when accessing these "pointers".

    // prop "points" to prop_info structure if there is a propery associated with the trie node.
    // Its situation is similar to the left, right, children "pointers". So we use
    // atomic_uint_least32_t and release-consume ordering to protect it as well.

    // We should also avoid rereading these fields redundantly, since not
    // all processor implementations ensure that multiple loads from the
    // same field are carried out in the right order.
    atomic_uint_least32_t prop;

    atomic_uint_least32_t left;
    atomic_uint_least32_t right;

    atomic_uint_least32_t children;

    char name[0];

    prop_bt(const char *name, const uint8_t name_length) {
        this->namelen = name_length;
        memcpy(this->name, name, name_length);
        this->name[name_length] = '\0';
    }

private:
    DISALLOW_COPY_AND_ASSIGN(prop_bt);
};

//属性的数据结构
struct prop_info {
    atomic_uint_least32_t serial;
    char value[PROP_VALUE_MAX];
    char name[0];

    prop_info(const char *name, const uint8_t namelen, const char *value,
              const uint8_t valuelen) {
        memcpy(this->name, name, namelen);
        this->name[namelen] = '\0';
        atomic_init(&this->serial, valuelen << 24);
        memcpy(this->value, value, valuelen);
        this->value[valuelen] = '\0';
    }
private:
    DISALLOW_COPY_AND_ASSIGN(prop_info);
};
```

###查找一个属性
```c++
const prop_info *__system_property_find(const char *name)
{
    if (__predict_false(compat_mode)) {
        return __system_property_find_compat(name);
    }
    return find_property(root_node(), name, strlen(name), NULL, 0, false);
}

//获取跟节点
static inline prop_bt *root_node()
{
    return reinterpret_cast<prop_bt*>(to_prop_obj(0));
}

//定位带某个节点
static void *to_prop_obj(uint_least32_t off)
{
    if (off > pa_data_size)
        return NULL;
    if (!__system_property_area__)
        return NULL;

    return (__system_property_area__->data + off);
}

static const prop_info *find_property(prop_bt *const trie, const char *name,
        uint8_t namelen, const char *value, uint8_t valuelen,
        bool alloc_if_needed)
{
    if (!trie) return NULL;

    const char *remaining_name = name;
    prop_bt* current = trie;
    while (true) {
        const char *sep = strchr(remaining_name, '.');
        const bool want_subtree = (sep != NULL);
        const uint8_t substr_size = (want_subtree) ?
            sep - remaining_name : strlen(remaining_name);

        if (!substr_size) {
            return NULL;
        }

        prop_bt* root = NULL;
        uint_least32_t children_offset = atomic_load_explicit(&current->children, memory_order_relaxed);
        if (children_offset != 0) {
            root = to_prop_bt(&current->children);
        } else if (alloc_if_needed) {
            uint_least32_t new_offset;
            root = new_prop_bt(remaining_name, substr_size, &new_offset);
            if (root) {
                atomic_store_explicit(&current->children, new_offset, memory_order_release);
            }
        }

        if (!root) {
            return NULL;
        }

        current = find_prop_bt(root, remaining_name, substr_size, alloc_if_needed);
        if (!current) {
            return NULL;
        }

        if (!want_subtree)
            break;

        remaining_name = sep + 1;
    }

    uint_least32_t prop_offset = atomic_load_explicit(&current->prop, memory_order_relaxed);
    if (prop_offset != 0) {
        return to_prop_info(&current->prop);
    } else if (alloc_if_needed) {
        uint_least32_t new_offset;
        prop_info* new_info = new_prop_info(name, namelen, value, valuelen, &new_offset);
        if (new_info) {
            atomic_store_explicit(&current->prop, new_offset, memory_order_release);
        }

        return new_info;
    } else {
        return NULL;
    }
}
```

#property_init

property_init的职责就是初始化客户端进程的属性存储区域，代码分析：
```C++
void property_init() {
    //第一次调用property_area_initialized为false
    if (property_area_initialized) {
        return;
    }

    property_area_initialized = true;

    if (__system_property_area_init()) {
        return;
    }
    /*
    pa_workspace是一个全局变量，其数据结构为：
    static workspace pa_workspace;
    struct workspace {
        size_t size;
        int fd;
    };
    #define PROP_FILENAME "/dev/__properties__"
    */
    pa_workspace.size = 0;
    pa_workspace.fd = open(PROP_FILENAME, O_RDONLY | O_NOFOLLOW | O_CLOEXEC);
    if (pa_workspace.fd == -1) {
        ERROR("Failed to open %s: %s\n", PROP_FILENAME, strerror(errno));
        return;
    }
}
int __system_property_area_init()
{
    return map_prop_area_rw();
}
static int map_prop_area_rw()
{
    /* dev is a tmpfs that we can use to carve a shared workspace
     * out of, so let's do that...
     * property_filename定义：
     * static char property_filename[PATH_MAX] = PROP_FILENAME;
     * #define PROP_FILENAME "/dev/__properties__"
     */
    const int fd = open(property_filename,
                        O_RDWR | O_CREAT | O_NOFOLLOW | O_CLOEXEC | O_EXCL, 0444);

    if (fd < 0) {
        if (errno == EACCES) {
            /* for consistency with the case where the process has already
             * mapped the page in and segfaults when trying to write to it
             */
            abort();
        }
        return -1;
    }
    //将/dev/__properties__的大小设置为PA_SIZE,便于mmap
    //#define PA_SIZE         (128 * 1024)
    if (ftruncate(fd, PA_SIZE) < 0) {
        close(fd);
        return -1;
    }

    pa_size = PA_SIZE;
    pa_data_size = pa_size - sizeof(prop_area);
    compat_mode = false;
    //创建一块共享内存
    void *const memory_area = mmap(NULL, pa_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (memory_area == MAP_FAILED) {
        close(fd);
        return -1;
    }
    //创建一个prop_area的对象，对象的处于上面mmap出来的共享内存当中。
    prop_area *pa = new(memory_area) prop_area(PROP_AREA_MAGIC, PROP_AREA_VERSION);

    /* plug into the lib property services */
    __system_property_area__ = pa;

    close(fd);
    return 0;
}
```



-   property_set
设置一个属性值，即往初始化好的属性区域写数据。代码分析：

```
//这只是个接口，真正的实现在一个static方法property_set_impl中，接口和实现分开
int property_set(const char* name, const char* value) {
    int rc = property_set_impl(name, value);
    if (rc == -1) {
        ERROR("property_set(\"%s\", \"%s\") failed\n", name, value);
    }
    return rc;
}
static int property_set_impl(const char* name, const char* value) {
    size_t namelen = strlen(name);
    size_t valuelen = strlen(value);

    //属性名字的合法性进行判断，如长度，是否有特殊字符等
    if (!is_legal_property_name(name, namelen)) return -1;
    if (valuelen >= PROP_VALUE_MAX) return -1;

    //判断属性是否和selinux相关，并采取相关操作，详见selinux章节
    if (strcmp("selinux.reload_policy", name) == 0 && strcmp("1", value) == 0) {
        if (selinux_reload_policy() != 0) {
            ERROR("Failed to reload policy\n");
        }
    } else if (strcmp("selinux.restorecon_recursive", name) == 0 && valuelen > 0) {
        if (restorecon_recursive(value) != 0) {
            ERROR("Failed to restorecon_recursive %s\n", value);
        }
    }

    prop_info* pi = (prop_info*) __system_property_find(name);

    if(pi != 0) {
        /* ro.* properties may NEVER be modified once set */
        if(!strncmp(name, "ro.", 3)) return -1;

        __system_property_update(pi, value, valuelen);
    } else {
        int rc = __system_property_add(name, namelen, value, valuelen);
        if (rc < 0) {
            return rc;
        }
    }
    /* If name starts with "net." treat as a DNS property. */
    if (strncmp("net.", name, strlen("net.")) == 0)  {
        if (strcmp("net.change", name) == 0) {
            return 0;
        }
       /*
        * The 'net.change' property is a special property used track when any
        * 'net.*' property name is updated. It is _ONLY_ updated here. Its value
        * contains the last updated 'net.*' property.
        */
        property_set("net.change", name);
    } else if (persistent_properties_loaded &&
            strncmp("persist.", name, strlen("persist.")) == 0) {
        /*
         * Don't write properties to disk until after we have read all default properties
         * to prevent them from being overwritten by default values.
         */
        write_persistent_property(name, value);
    }
    property_changed(name, value);
    return 0;
}
```
