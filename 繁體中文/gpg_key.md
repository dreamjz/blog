---
title: "為 Git commit 添加簽名"
date: 2023-11-22
category:
 - Git
---

## 1. PGP、OpenPGP、GnuPG 和 gpg

- PGP(Pretty Good Privacy)：最初的商業軟件名
- OpenPGP： PGP 標準
- GnuPG(Gnu Privacy Guard)：實現了 OpenPGP 的軟件
- gpg：GnuPG 的命令行工具

## 2. Git 設計的缺陷

Git commit 中的 author 中的`user.name`和`user.email`可以任意設置，沒有進行任何的校驗。

可以通過`filer-branch`批量修改整個 Git 倉庫的歷史：

```sh
git commit -am "Destroy production"
git filter-branch --env-filter \
  'if [ "$GIT_AUTHOR_EMAIL" = "iamthe@evilguy.com" ]; then
     GIT_AUTHOR_EMAIL="unsuspecting@victim.com";
     GIT_AUTHOR_NAME="Unsuspecting Victim";
     GIT_COMMITTER_EMAIL=$GIT_AUTHOR_EMAIL;
     GIT_COMMITTER_NAME="$GIT_AUTHOR_NAME"; fi' -- --all
git push -f
```

## 3. 為 Git commit 添加簽名

### 3.1 作用

- 防止惡意偽造 Git commit，參見[震惊！竟然有人在 GitHub 上冒充我的身份！      ](https://blog.spencerwoo.com/2020/08/wait-this-is-not-my-commit/)
- <del>GitHub 上帶有 Verified 標識，與眾不同</del>

### 3.2 安裝 gpg

```sh
# windows
$ scoop install gpg

# linux 一般自帶
# 沒有的話使用對應的包管理器安裝即可
# arch
$ pacman -S gpg

$ gpg --version 
gpg (GnuPG) 2.4.3
libgcrypt 1.10.2
Copyright (C) 2023 g10 Code GmbH
...
```

### 3.3 生成 gpg 密鑰對(公鑰和私鑰)

```sh
$ gpg --full-generate-key
```

- 密鑰種類：選擇默認的 RSA 即可
- 密鑰長度：選擇 GitHub 推薦的 4096 bits
- 密鑰過期時間：根據自己的需要選擇
- 輸入用戶名和郵箱
- 設置安全密碼，**密碼一定要記住**

### 3.4 查看密鑰

```sh
$ gpg --list-secret-keys --keyid-format=long
...
sec   rsa4096/24CD550268849CA0 2020-08-29 [SC]
      9433E1B6807DE7C15E20DC3B24CD550268849CA0
uid                 [ultimate] Spencer Woo (My GPG key) <my@email.com>
ssb   rsa4096/EB754D2B2409E9FE 2020-08-29 [E]
```

其中的`24CD550268849CA0`就是私鑰的 ID。

### 3.5 為 Git commit 添加簽名

```sh
# 配置 gpg 程序
$ git config --global gpg.program $(which gpg)
# 指定簽名用的密鑰
$ git config --global user.signingkey 24CD550268849CA0
# commit 時自動簽名
$ git config --global commit.gpgsign true
```

commit 一次，之後查看 git log

```sh
$ git log --show-signature
commit 1c4a03ba8a9629d02913406099d03a5ff1aa200d (HEAD -> test)
gpg: Signature made Wed Nov 22 02:39:51 2023 CST
gpg:                using RSA key xxx
gpg: Good signature from "xxx" [ultimate]
```

### 3.6 導出公鑰並添加到 GitHub 賬戶中

```sh
# 獲取私鑰 ID
$ gpg --list-secret-keys --keyid-format=long
...
sec   rsa4096/24CD550268849CA0 2020-08-29 [SC]
...
# 生成公鑰
$ gpg --armor --export 24CD550268849CA0
...
```

將輸出的內容粘貼至 [Settings » SSH and GPG keys » New GPG key](https://github.com/settings/keys) 。

## 4. 遷移 gpg key

有時需要在不同的開發環境中使用相同的 gpg 配置，此時就需要進行備份和遷移。

以 Windows 遷移到 ArchLinux 為例。

### 4.1 導出所需的 key

```sh
# 導出密鑰對到文件中
$ gpg --export-secret-keys -a <keyid> > private_key.asc
$ gpg --export -a <keyid> > public_key.asc
```

### 4.2 導出 trust db 

```sh
gpg --export-ownertrust > file.txt
```

若忽略這一步，在另一環境中，gpg 將提示密鑰 userid 為 unknown 狀態。

### 4.3 導入 key 和 trust db

```sh
# 導入 trust
$ gpg --import-ownertrust < file.txt

# 導入密鑰
gpg --import private_key.asc
gpg --import public_key.asc
```

### 4.4 配置 Git

```sh
# 指定簽名用的密鑰
$ git config --global user.signingkey <keyid>
# commit 時自動簽名
$ git config --global commit.gpgsign true
```

若出現了` gpg failed to sign the data`

在對應 shell 的 profile 文件中添加（以 zsh 為例）：

```sh
# ~/.zprofile
export GPG_TTY=$(tty)
```

## 參考

1. https://en.wikipedia.org/wiki/Pretty_Good_Privacy
2. [用更现代的方法使用PGP](https://ulyc.github.io/2021/01/13/2021%E5%B9%B4-%E7%94%A8%E6%9B%B4%E7%8E%B0%E4%BB%A3%E7%9A%84%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8PGP-%E4%B8%8A/)
3. [震惊！竟然有人在 GitHub 上冒充我的身份！      ](https://blog.spencerwoo.com/2020/08/wait-this-is-not-my-commit/)
4. [迁移 GPG 密钥](https://anandzhang.com/posts/essay/5)
5. [How to migrate GPG trust database from one machine to another?](https://unix.stackexchange.com/questions/210348/how-to-migrate-gpg-trust-database-from-one-machine-to-another)
6. [Managing commit signature verification](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification)