# Linuxシステムプログラミング入門 修了試験解答
## ログインしてみる
問題サーバにログインすると以下のようなREADMEがあります．
```
execute file /try_running_me and see what happens!
googling "unshare" may helps.
```
## ファイルを実行してみる
ルートディレクトリの`/try_running_me`を実行してみると，以下のように出力されます．
```
oops!
run this binary with setting 0-th argument to "" (empty string)
```
## execveを使って指定ファイルを実行
以下のようなコードを記述，コンパイルして，第一引数を空文字列にしてファイルを実行します．
```c
#define _GNU_SOURCE
#include <unistd.h>

#define FILE_PATH "/try_running_me"

int main(){
        char* arg[] = {"",NULL};
        execve(FILE_PATH,arg,environ);
}
```
すると以下のようなメッセージが表示されます．
```
you're half way there!
run this binary as root. note that you should keep 0-th argument ""
```
メッセージによると，rootでこのファイルを実行する必要があります．
## 解法１：unshareを使う
`unshare`というコマンドを利用します．`-r`というオプションを利用すると`root`になることができます．
```
unshare -r ./a.out
```
## 解法２：cloneでCで記述する
以下のようなCコードを記述すると，cloneで名前空間を利用することができます．
講義では`pipe`を使って子プロセスと同期を取っていましたが，`sleep`で子プロセスで十分な時間待つことで簡略化しています．
```c
#define _GNU_SOURCE
#include <unistd.h>
#include <sched.h>
#include <sys/wait.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>

#define FILE_PATH "/try_running_me"

int child(void* arg){
        sleep(1);
        char* ap[] = {"",NULL};
        execve(FILE_PATH,ap,environ);
}

int wf(char* s,char* p){
        int fd = open(p,O_WRONLY);
        int len = 0;
        for(len = 0;s[len] != '\0';len++);
        write(fd,s,len);
        close(fd);
}

int ug(pid_t c,uid_t uid){
        char path[1024],data[1024];
        snprintf(path,1024,"/proc/%d/uid_map",c);
        snprintf(data,1024,"0 %d 1\n",uid);
        return wf(data,path);
}

int main(){
        void* stack = mmap(NULL,1024*1024,PROT_READ|PROT_WRITE,
                            MAP_PRIVATE|MAP_ANONYMOUS|MAP_GROWSDOWN|MAP_STACK,-1,0);
        pid_t c = clone(child,stack+1024*1024,SIGCHLD|CLONE_NEWUSER,stack);
        ug(c,getuid());
        waitpid(c,NULL,0);
}
```
## 実行・フラグ取得
以上のいずれかの方法でrootで実行すると，フラグを獲得することができます．
```
hooray! you finally made it!
your flag is: mc2021{y0u_l1nux_h4ck3r}
```
