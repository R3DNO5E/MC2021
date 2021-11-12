# Linuxシステムプログラミング入門 演習フォローアップ資料
困ったときのための演習の答えです．
## プロジェクトの準備
以下のようなディレクトリ構成を作る
```
~/camp_container/
    - src
        - main.c
    - build
    - CMakeLists.txt
```
CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.16)
project(camp_container)
set(CMAKE_C_STANDARD 11)

add_executable(camp_container src/main.c)
```
main.c
```c
#define _GNU_SOURCE

int main(){
    
}
```
## initを作る
```shell
#! /bin/bash
source /etc/profile
mount -t proc proc /proc
mount -t sysfs sysfs /sysfs
set -m
/bin/bash --login
umount /proc
umount /sys
```
## chrootしてディレクトリを変更する関数を書く
```c
int chroot_dir(const char*const path){
	if(chroot(path) != 0) return -1;
	if(chdir(“/”) != 0) return -1;
	return 0;
}
```
## 子プロセス作成とinit実行
```c
#define _GNU_SOURCE

#include <unistd.h>
#include <sched.h>
#include <sys/mman.h>
#include <sys/wait.h>

#define SYSROOT_DIR "/home/minicamp/sysroot-debian-bullseye"
#define INIT_PATH "/bin/init"
#define STACK_SIZE (16*1024*1024)

int chroot_dir(const char *const path) {
    if (chroot(path) != 0) return -1;
    if (chdir("/") != 0) return -1;
    return 0;
}

typedef struct {
} isolated_child_args_t;

int isolated_child(isolated_child_args_t *args) {
    if (chroot_dir(SYSROOT_DIR) == -1)return -1;
    char *const init_arg[] = {INIT_PATH, NULL};
    char *const init_env[] = {NULL};
    execve(INIT_PATH, init_arg, init_env);
    return -1;
}

int start_child() {
    isolated_child_args_t args;
    char *stack = mmap(NULL, STACK_SIZE, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_GROWSDOWN | MAP_STACK, -1, 0);
    if (stack == MAP_FAILED) return -1;
    pid_t child = clone((int (*)(void *)) isolated_child, stack + STACK_SIZE, SIGCHLD, &args);
    if (child == -1) return -1;
    if (waitpid(child, NULL, 0) == -1) return -1;
    return 0;
}

int main(){
    return start_child();
}
```
## ファイルに何か書き込む関数を書く
```c
int write_file(const char *const path, const char *const str) {
    int fd = open(path, O_WRONLY), len;
    if (fd == -1)return -1;
    for (len = 0; str[len] != '\0'; len++);
    if (write(fd, str, len) != len)return -1;
    if (close(fd) == -1) return -1;
    return 0;
}
```
## マッピングを書き込む関数を書く
```c
int parent_write_ug_map(pid_t child) {
    char path[STR_BUF_SIZE], data[STR_BUF_SIZE];
    snprintf(path, STR_BUF_SIZE, "/proc/%d/setgroups", child);
    if (write_file(path, "deny") == -1) return -1;
    snprintf(path, STR_BUF_SIZE, "/proc/%d/gid_map", child);
    snprintf(data, STR_BUF_SIZE, "0 %d 1\n", getgid());
    if (write_file(path, data) == -1) return -1;
    snprintf(path, STR_BUF_SIZE, "/proc/%d/uid_map", child);
    snprintf(data, STR_BUF_SIZE, "0 %d 1\n", getuid());
    if (write_file(path, data) == -1) return -1;
    return 0;
}
```
## 親プロセスが準備している間待つようにする
```c
typedef struct {
    int fd[2];
} isolated_child_args_t;

int isolated_child(isolated_child_args_t *args) {
    char buf[1];
    if (read(args->fd[0], buf, 1) == -1)return -1;
    if (chroot_dir(SYSROOT_DIR) == -1)return -1;
    char *const init_arg[] = {INIT_PATH, NULL};
    char *const init_env[] = {NULL};
    execve(INIT_PATH, init_arg, init_env);
    return -1;
}

int start_child() {
    isolated_child_args_t args;
    if (pipe(args.fd) == -1) return -1;
    char *stack = mmap(NULL, STACK_SIZE, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_GROWSDOWN | MAP_STACK, -1, 0);
    if (stack == MAP_FAILED) return -1;
    pid_t child = clone((int (*)(void *)) isolated_child, stack + STACK_SIZE,
                        SIGCHLD | CLONE_NEWPID | CLONE_NEWUSER | CLONE_NEWNS, &args);
    if (child == -1) return -1;
    if (parent_write_ug_map(child) == -1) return -1;
    if (write(args.fd[1], "\0", 1) == -1) return -1;
    if (waitpid(child, NULL, 0) == -1) return -1;
    return 0;
}
```
## コマンドを実行する関数を書く
```c
int exec_command_child(void *arg) {
    char *const argv[] = {SHELL_PATH, "-c", arg, NULL}, *const envp[] = {NULL};
    execve(SHELL_PATH, argv, envp);
    return -1;
}

int exec_command(const char *const cmd) {
    char *stack = mmap(NULL, STACK_SIZE, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_GROWSDOWN | MAP_STACK, -1, 0);
    if (stack == MAP_FAILED) return -1;
    pid_t child = clone((int (*)(void *)) exec_command_child, stack + STACK_SIZE, SIGCHLD, cmd);
    if (child == -1)return -1;
    if (waitpid(child, NULL, 0) == -1) return -1;
    return 0;
}
```
## ネットワークの設定(親プロセス)
```c
int parent_install_network(pid_t child) {
    exec_command("ip link add vethA type veth peer vethB");
    exec_command("ip link set dev vethA up");
    exec_command( "ip address add 10.0.0.1/24 dev vethA");
    char cmd[STR_BUF_SIZE];
    snprintf(cmd, STR_BUF_SIZE, "ip link set dev vethB netns %d",child);
    exec_command(cmd);
}
```
## ネットワークの設定(子プロセス)
```c
int child_install_network() {
    exec_command( "ip link set dev vethB up");
    exec_command( "ip address add 10.0.0.2/24 dev vethB");
    exec_command("ip route add default via 10.0.0.1");
}
```
## ネットワークの片付け(親プロセス)
```c
int parent_uninstall_network() {
    exec_command( "ip link delete vethA");
}
```
## ネットワーク関連の処理を実行するようにする
```c
int isolated_child(isolated_child_args_t *args) {
    char buf[1];
    if (read(args->fd[0], buf, 1) == -1)return -1;
    if (chroot_dir(SYSROOT_DIR) == -1)return -1;
    if (child_install_network() == -1) return -1;
    char *const init_arg[] = {INIT_PATH, NULL};
    char *const init_env[] = {NULL};
    execve(INIT_PATH, init_arg, init_env);
    return -1;
}

int start_child() {
    isolated_child_args_t args;
    if (pipe(args.fd) == -1) return -1;
    char *stack = mmap(NULL, STACK_SIZE, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_GROWSDOWN | MAP_STACK, -1, 0);
    if (stack == MAP_FAILED) return -1;
    pid_t child = clone((int (*)(void *)) isolated_child, stack + STACK_SIZE,
                        SIGCHLD | CLONE_NEWPID | CLONE_NEWUSER | CLONE_NEWNET, &args);
    if (child == -1) return -1;
    if (parent_write_ug_map(child) == -1) return -1;
    if (parent_install_network(child) == -1) return -1;
    if (write(args.fd[1], "\0", 1) == -1) return -1;
    if (waitpid(child, NULL, 0) == -1) return -1;
    if (parent_uninstall_network() == -1) return -1;
    return 0;
}
```
