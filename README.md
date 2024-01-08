## A. Building
#### 1. Clone this repository
```sh
git clone https://github.com/rsuntk/android_kernel_samsung_a12s-4.19-rebased.git rebaseda12s && cd rebaseda12s
```
#### 2. Get required Toolchains:
- **For Galaxy A12s you need:**
  - [clang-r353983c](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/emu-29.0-release/clang-r353983c.tar.gz)
  - [aarch64-linux-android](https://github.com/growtopiajaw/aarch64-linux-android-4.9)
  - [aarch64-linux-gnu](https://github.com/radcolor/aarch64-linux-gnu)
#### 2. Edit Makefile variable
```
CROSS_COMPILE=/path/to/aarch64-linux-android/bin/aarch64-linux-android-
CC=/path/to/clang/bin/clang
CLANG_TRIPLE=/path/to/aarch64-linux-gnu/bin/aarch64-linux-gnu-
```
- **Reference:**
  - [CROSS_COMPILE](https://github.com/rsuntk/android_kernel_samsung_a12s-4.19-rebased/blob/android-4.19-stable/Makefile#L323)
  - [CC](https://github.com/rsuntk/android_kernel_samsung_a12s-4.19-rebased/blob/android-4.19-stable/Makefile#L374)
  - [CLANG_TRIPLE](https://github.com/rsuntk/android_kernel_samsung_a12s-4.19-rebased/blob/android-4.19-stable/Makefile#L494)
#### 3. Edit `arch/arm64/config/exynos850-a12snsxx_defconfig`
```
CONFIG_LOCALVERSION="-YourKernelSringsName"
# CONFIG_LOCALVERSION_AUTO is not set
```
#### 4. Get this [build script](https://github.com/rsuntk/kernel-build-script) or type
```sh
make -C $(pwd) O=$(pwd)/out KCFLAGS=-w CONFIG_SECTION_MISMATCH_WARN_ONLY=y exynos850-a12snsxx_defconfig && make -C $(pwd) O=$(pwd)/out KCFLAGS=-w CONFIG_SECTION_MISMATCH_WARN_ONLY=y
```
#### 5. Check directory out/arch/arm64/boot
```sh
cd $(pwd)/out/arch/arm64/boot && ls
Image.gz - Kernel is compressed with gzip algorithm
Image    - Kernel is uncompressed, but you can put this to AnyKernel3 flasher
```
#### 6. Put Image.gz/Image to Anykernel3 zip, don't forget to modify the boot partition path in anykernel.sh
#### 7. Done, enjoy.
## B. How to add [KernelSU](https://kernelsu.org) support
#### 1. First, add KernelSU to your kernel source tree:
```sh
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -
```
#### 2. You can use KPROBES, but Manual method are more stable. Edit ```arch/arm64/configs/exynos850-a12snsxx_defconfig```, and the edit these lines
**From this:**
```
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBE_EVENTS=y
```
**To this:**
```
# CONFIG_KPROBES is not set
# CONFIG_HAVE_KPROBES is not set
# CONFIG_KPROBE_EVENTS is not set
```
#### 3. Then add KernelSU config line to ```arch/arm64/configs/exynos850-a12snsxx_defconfig```
```
CONFIG_KSU=y
# CONFIG_KSU_DEBUG is not set # if you a dev, then turn on this option for KernelSU debugging.
```
#### 4. Edit these file:
- **fs/exec.c**
```diff
diff --git a/fs/exec.c b/fs/exec.c
index ac59664eaecf..bdd585e1d2cc 100644
--- a/fs/exec.c
+++ b/fs/exec.c
@@ -1890,11 +1890,14 @@ static int __do_execve_file(int fd, struct filename *filename,
 	return retval;
 }

+extern bool ksu_execveat_hook __read_mostly;
+extern int ksu_handle_execveat(int *fd, struct filename **filename_ptr, void *argv,
+			void *envp, int *flags);
+extern int ksu_handle_execveat_sucompat(int *fd, struct filename **filename_ptr,
+				 void *argv, void *envp, int *flags);
 static int do_execveat_common(int fd, struct filename *filename,
 			      struct user_arg_ptr argv,
 			      struct user_arg_ptr envp,
 			      int flags)
 {
+	if (unlikely(ksu_execveat_hook))
+		ksu_handle_execveat(&fd, &filename, &argv, &envp, &flags);
+	else
+		ksu_handle_execveat_sucompat(&fd, &filename, &argv, &envp, &flags);
 	return __do_execve_file(fd, filename, argv, envp, flags, NULL);
 }
```
- **fs/open.c**
```diff
diff --git a/fs/open.c b/fs/open.c
index 05036d819197..965b84d486b8 100644
--- a/fs/open.c
+++ b/fs/open.c
@@ -348,6 +348,8 @@ SYSCALL_DEFINE4(fallocate, int, fd, int, mode, loff_t, offset, loff_t, len)
 	return ksys_fallocate(fd, mode, offset, len);
 }

+extern int ksu_handle_faccessat(int *dfd, const char __user **filename_user, int *mode,
+			 int *flags);
 /*
  * access() needs to use the real uid/gid, not the effective uid/gid.
  * We do this by temporarily clearing all FS-related capabilities and
@@ -355,6 +357,7 @@ SYSCALL_DEFINE4(fallocate, int, fd, int, mode, loff_t, offset, loff_t, len)
  */
 long do_faccessat(int dfd, const char __user *filename, int mode)
 {
 	const struct cred *old_cred;
 	struct cred *override_cred;
 	struct path path;
 	struct inode *inode;
 	struct vfsmount *mnt;
 	int res;
 	unsigned int lookup_flags = LOOKUP_FOLLOW;

+	ksu_handle_faccessat(&dfd, &filename, &mode, NULL);

 	if (mode & ~S_IRWXO)	/* where's F_OK, X_OK, W_OK, R_OK? */
 		return -EINVAL;
```
- **fs/read_write.c**
```diff
diff --git a/fs/read_write.c b/fs/read_write.c
index 650fc7e0f3a6..55be193913b6 100644
--- a/fs/read_write.c
+++ b/fs/read_write.c
@@ -434,10 +434,14 @@ ssize_t kernel_read(struct file *file, void *buf, size_t count, loff_t *pos)
 }
 EXPORT_SYMBOL(kernel_read);

+extern bool ksu_vfs_read_hook __read_mostly;
+extern int ksu_handle_vfs_read(struct file **file_ptr, char __user **buf_ptr,
+			size_t *count_ptr, loff_t **pos);
 ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
 {
 	ssize_t ret;

+	if (unlikely(ksu_vfs_read_hook))
+		ksu_handle_vfs_read(&file, &buf, &count, &pos);
+
 	if (!(file->f_mode & FMODE_READ))
 		return -EBADF;
 	if (!(file->f_mode & FMODE_CAN_READ))
```
- **fs/stat.c**
```diff
diff --git a/fs/stat.c b/fs/stat.c
index 376543199b5a..82adcef03ecc 100644
--- a/fs/stat.c
+++ b/fs/stat.c
@@ -148,6 +148,8 @@ int vfs_statx_fd(unsigned int fd, struct kstat *stat,
 }
 EXPORT_SYMBOL(vfs_statx_fd);

+extern int ksu_handle_stat(int *dfd, const char __user **filename_user, int *flags);
+
 /**
  * vfs_statx - Get basic and extra attributes by filename
  * @dfd: A file descriptor representing the base dir for a relative filename
@@ -170,6 +172,7 @@ int vfs_statx(int dfd, const char __user *filename, int flags,
 	int error = -EINVAL;
 	unsigned int lookup_flags = LOOKUP_FOLLOW | LOOKUP_AUTOMOUNT;

+	ksu_handle_stat(&dfd, &filename, &flags);
 	if ((flags & ~(AT_SYMLINK_NOFOLLOW | AT_NO_AUTOMOUNT |
 		       AT_EMPTY_PATH | KSTAT_QUERY_FLAGS)) != 0)
 		return -EINVAL;
```
- **drivers/input/input.c**
```diff
diff --git a/drivers/input/input.c b/drivers/input/input.c
index 45306f9ef247..815091ebfca4 100755
--- a/drivers/input/input.c
+++ b/drivers/input/input.c
@@ -367,10 +367,13 @@ static int input_get_disposition(struct input_dev *dev,
 	return disposition;
 }

+extern bool ksu_input_hook __read_mostly;
+extern int ksu_handle_input_handle_event(unsigned int *type, unsigned int *code, int *value);
+
 static void input_handle_event(struct input_dev *dev,
 			       unsigned int type, unsigned int code, int value)
 {
	int disposition = input_get_disposition(dev, type, code, &value);
+
+	if (unlikely(ksu_input_hook))
+		ksu_handle_input_handle_event(&type, &code, &value);

 	if (disposition != INPUT_IGNORE_EVENT && type != EV_SYN)
 		add_input_randomness(type, code, value);
```
- **See full KernelSU non-GKI integration documentations** [here](https://kernelsu.org/guide/how-to-integrate-for-non-gki.html)
## C. Credit
- [Physwizz](https://github.com/physwizz) - OEM and Permissive kernel source
- [Rissu](https://github.com/rsuntk) - Rebased kernel source
- [KernelSU](https://kernelsu.org) - A kernel-based root solution for Android
