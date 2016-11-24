---
date: 2016-11-22 14:05
title: Linker简介
tags: [android]
categories: [android,linker]
---

##linker初始化
###_start
linke的入口函数, 该函数调用了linker_init，并传入一个参数sp(堆栈指针)
```
ENTRY(_start)
  mov x0, sp
  bl __linker_init

  /* linker init returns the _entry address in the main image */
  br x0
END(_start)
```

###__linker_init
负责初始化 linker，完成 linker 的重定位工作。
```
extern "C" ElfW(Addr) __linker_init(void* raw_args) {
  KernelArgumentBlock args(raw_args);

  //linker 镜像的开始地址
  ElfW(Addr) linker_addr = args.getauxval(AT_BASE);

  //linker的入口点，禁止shell中调用linker
  ElfW(Addr) entry_point = args.getauxval(AT_ENTRY);
  ElfW(Ehdr)* elf_hdr = reinterpret_cast<ElfW(Ehdr)*>(linker_addr);
  ElfW(Phdr)* phdr = reinterpret_cast<ElfW(Phdr)*>(linker_addr + elf_hdr->e_phoff);

  soinfo linker_so(nullptr, nullptr, 0, 0);

  // If the linker is not acting as PT_INTERP entry_point is equal to
  // _start. Which means that the linker is running as an executable and
  // already linked by PT_INTERP.
  //
  // This happens when user tries to run 'adb shell /system/bin/linker'
  // see also https://code.google.com/p/android/issues/detail?id=63174
  if (reinterpret_cast<ElfW(Addr)>(&_start) == entry_point) {
    __libc_fatal("This is %s, the helper program for shared library executables.\n", args.argv[0]);
  }

  linker_so.base = linker_addr;
  linker_so.size = phdr_table_get_load_size(phdr, elf_hdr->e_phnum);
  linker_so.load_bias = get_elf_exec_load_bias(elf_hdr);
  linker_so.dynamic = nullptr;
  linker_so.phdr = phdr;
  linker_so.phnum = elf_hdr->e_phnum;
  linker_so.set_linker_flag();

  // This might not be obvious... The reasons why we pass g_empty_list
  // in place of local_group here are (1) we do not really need it, because
  // linker is built with DT_SYMBOLIC and therefore relocates its symbols against
  // itself without having to look into local_group and (2) allocators
  // are not yet initialized, and therefore we cannot use linked_list.push_*
  // functions at this point.
  //linker的重定向工作
  if (!(linker_so.prelink_image() && linker_so.link_image(g_empty_list, g_empty_list, nullptr))) {
    // It would be nice to print an error message, but if the linker
    // can't link itself, there's no guarantee that we'll be able to
    // call write() (because it involves a GOT reference). We may as
    // well try though...
    const char* msg = "CANNOT LINK EXECUTABLE: ";
    write(2, msg, strlen(msg));
    write(2, __linker_dl_err_buf, strlen(__linker_dl_err_buf));
    write(2, "\n", 1);
    _exit(EXIT_FAILURE);
  }

  __libc_init_tls(args);

  // Initialize the linker's own global variables
  linker_so.call_constructors();

  // Initialize static variables. Note that in order to
  // get correct libdl_info we need to call constructors
  // before get_libdl_info().
  solist = get_libdl_info();
  sonext = get_libdl_info();

  // We have successfully fixed our own relocations. It's safe to run
  // the main part of the linker now.
  args.abort_message_ptr = &g_abort_message;
  ElfW(Addr) start_address = __linker_init_post_relocation(args, linker_addr);

  INFO("[ jumping to _start ]");

  // Return the address that the calling assembly stub should jump to.
  return start_address;
}

```


###__linker_init_post_relocation
初始化property, 连接debugd，预加载一下系统库
```
static ElfW(Addr) __linker_init_post_relocation(KernelArgumentBlock& args, ElfW(Addr) linker_base) {

  ...

  __system_properties_init();
  debuggerd_init();

  ...

  const char* ldpath_env = nullptr;
  const char* ldpreload_env = nullptr;
  if (!getauxval(AT_SECURE)) {
    ldpath_env = getenv("LD_LIBRARY_PATH");
    ldpreload_env = getenv("LD_PRELOAD");
  }

  ...

  soinfo* si = soinfo_alloc(args.argv[0], nullptr, 0, RTLD_GLOBAL);

  ...

  parse_LD_LIBRARY_PATH(ldpath_env);
  parse_LD_PRELOAD(ldpreload_env);

  somain = si;

  ...

  for (const auto& ld_preload_name : g_ld_preload_names) {
    needed_library_name_list.push_back(ld_preload_name.c_str());
    ++needed_libraries_count;
    ++ld_preloads_count;
  }

  for_each_dt_needed(si, [&](const char* name) {
    needed_library_name_list.push_back(name);
    ++needed_libraries_count;
  });

  const char* needed_library_names[needed_libraries_count];

  memset(needed_library_names, 0, sizeof(needed_library_names));
  needed_library_name_list.copy_to_array(needed_library_names, needed_libraries_count);

  if (needed_libraries_count > 0 &&
      !find_libraries(si, needed_library_names, needed_libraries_count, nullptr,
          &g_ld_preloads, ld_preloads_count, RTLD_GLOBAL, nullptr)) {
    __libc_format_fd(2, "CANNOT LINK EXECUTABLE: %s\n", linker_get_error_buffer());
    exit(EXIT_FAILURE);
  } else if (needed_libraries_count == 0) {
    if (!si->link_image(g_empty_list, soinfo::soinfo_list_t::make_list(si), nullptr)) {
      __libc_format_fd(2, "CANNOT LINK EXECUTABLE: %s\n", linker_get_error_buffer());
      exit(EXIT_FAILURE);
    }
    si->increment_ref_count();
  }

 ...

  {
    ProtectedDataGuard guard;

    si->call_pre_init_constructors();
    ...
    si->call_constructors();
  }


  TRACE("[ Ready to execute '%s' @ %p ]", si->get_realpath(), reinterpret_cast<void*>(si->entry));
  return si->entry;
```

##动态库的加载过程
dlopen->dlopen_ext->do_dlopen
```
soinfo* do_dlopen(const char* name, int flags, const android_dlextinfo* extinfo) {
  if ((flags & ~(RTLD_NOW|RTLD_LAZY|RTLD_LOCAL|RTLD_GLOBAL|RTLD_NODELETE|RTLD_NOLOAD)) != 0) {
    DL_ERR("invalid flags to dlopen: %x", flags);
    return nullptr;
  }
  if (extinfo != nullptr) {
    if ((extinfo->flags & ~(ANDROID_DLEXT_VALID_FLAG_BITS)) != 0) {
      DL_ERR("invalid extended flags to android_dlopen_ext: 0x%" PRIx64, extinfo->flags);
      return nullptr;
    }
    if ((extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD) == 0 &&
        (extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET) != 0) {
      DL_ERR("invalid extended flag combination (ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET without "
          "ANDROID_DLEXT_USE_LIBRARY_FD): 0x%" PRIx64, extinfo->flags);
      return nullptr;
    }
  }

  ProtectedDataGuard guard;
  soinfo* si = find_library(name, flags, extinfo);
  if (si != nullptr) {
    si->call_constructors();
  }
  return si;
}
```
find_library->find_libraries
```
static bool find_libraries(soinfo* start_with, const char* const library_names[],
      size_t library_names_count, soinfo* soinfos[], std::vector<soinfo*>* ld_preloads,
      size_t ld_preloads_count, int rtld_flags, const android_dlextinfo* extinfo) {

   ...

  // Step 1: load and pre-link all DT_NEEDED libraries in breadth first order.
  for (LoadTask::unique_ptr task(load_tasks.pop_front());
      task.get() != nullptr; task.reset(load_tasks.pop_front())) {
    soinfo* si = find_library_internal(load_tasks, task->get_name(), rtld_flags, extinfo);
    ...
  }

  // Step 2: link libraries.
 ...
  bool linked = local_group.visit([&](soinfo* si) {
    if (!si->is_linked()) {
      if (!si->link_image(global_group, local_group, extinfo)) {
        return false;
      }
      si->set_linked();
    }

    return true;
  });

  ...

}
```

find_library_internal->load_library->load_library
```
static soinfo* load_library(int fd, off64_t file_offset,
                            LoadTaskList& load_tasks,
                            const char* name, int rtld_flags,
                            const android_dlextinfo* extinfo) {
 ...
  // Read the ELF header and load the segments.
  ElfReader elf_reader(realpath.c_str(), fd, file_offset, file_stat.st_size);
  if (!elf_reader.Load(extinfo)) {
    return nullptr;
  }

  soinfo* si = soinfo_alloc(realpath.c_str(), &file_stat, file_offset, rtld_flags);
  if (si == nullptr) {
    return nullptr;
  }
  si->base = elf_reader.load_start();
  si->size = elf_reader.load_size();
  si->load_bias = elf_reader.load_bias();
  si->phnum = elf_reader.phdr_count();
  si->phdr = elf_reader.loaded_phdr();

  if (!si->prelink_image()) {
    soinfo_free(si);
    return nullptr;
  }

  for_each_dt_needed(si, [&] (const char* name) {
    load_tasks.push_back(LoadTask::create(name, si));
  });

  return si;
}
```
