# GitHub Pages を使った RPM リポジトリ構築手順

## 概要

この手順は、rpmパッケージを作って、GitHub Pagesで公開する流れをまとめている。
以下の流れで実施する。

1. テスト用 RPM パッケージを作成する
2. `createrepo_c` で RPM リポジトリを作成する
3. GitHub Pages で公開する
4. Linux ホストから `dnf` で利用確認する
5. リポジトリの内容を更新したときの作業手順を整理する

---

## 動作確認した構成

### パッケージ作成やGitHub関連の作業をした環境
- macOS Tahoe 26.3.1
- Podman Desktop

### パッケージインストール下環境
- Rockey Linux 9.7

---

## 前提

- GitHub アカウントを持っている
- GitHub 上に公開用リポジトリを作成済みである
  - 例: `ytooyama/rpm-repo`
- Podman が利用可能である
- Linux ホスト側で `dnf` が利用可能である
- rpmパッケージはnoarchのものを作成

---

## 1. テスト用 RPM パッケージを作成する

今回は動作確認用として、`hello-rpm` という簡単な RPM を作成する。
この RPM は `/usr/local/bin/hello-rpm` というコマンドを配置し、実行するとメッセージを表示する。

### 1.1 作業ディレクトリを作成する

```bash
mkdir -p ~/tmp/hello-rpm
cd ~/tmp/hello-rpm
```

### 1.2 実行ファイルを作成する
ここら辺はテスト用の内容。すでに公開したいrpmパッケージがあるなら省略。

```bash
cat > hello-rpm <<'EOF'
#!/bin/bash
echo "Hello from hello-rpm"
EOF

chmod +x hello-rpm
```

### 1.3 SPEC ファイルを作成する
ここら辺はテスト用の内容。すでに公開したいrpmパッケージがあるなら省略。

```bash
cat > hello-rpm.spec <<'EOF'
Name:           hello-rpm
Version:        0.1.0
Release:        1%{?dist}
Summary:        Simple test RPM package
License:        MIT
BuildArch:      noarch

%description
Test RPM package

%prep
%build

%install
mkdir -p %{buildroot}/usr/local/bin
install -m 0755 %{_sourcedir}/hello-rpm %{buildroot}/usr/local/bin/hello-rpm

%files
/usr/local/bin/hello-rpm

%changelog
* Wed Mar 25 2026 Test <test@example.com> - 0.1.0-1
- Initial
EOF
```

### 1.4 Podman で RPM をビルドする
うっかり防止のために、Podmanコンテナで全部作業する。
ここら辺はテスト用の内容。すでに公開したいrpmパッケージがあるなら省略。

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:9 \
  bash -lc '
    dnf install -y rpm-build &&
    mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS} &&
    cp hello-rpm ~/rpmbuild/SOURCES/ &&
    cp hello-rpm.spec ~/rpmbuild/SPECS/ &&
    rpmbuild -bb ~/rpmbuild/SPECS/hello-rpm.spec &&
    cp ~/rpmbuild/RPMS/noarch/*.rpm /work/
  '
```

### 1.5 生成物を確認する
ここら辺はテスト用の内容。すでに公開したいrpmパッケージがあるなら省略。

```bash
ls -lh *.rpm
```

成功すると、例えば次のようなファイルができる。

```text
hello-rpm-0.1.0-1.el9.noarch.rpm
```

---

## 2. RPM リポジトリを作成する
ここから先は、GitHub Pagesでrpmパッケージを公開したいときに実行する手順。

### 2.1 RPM 配置用ディレクトリを作成する
公開したいrpmパッケージを適当なディレクトリーにすべてコピーする。

```bash
mkdir -p rpm
mv hello-rpm-*.rpm rpm/
```

### 2.2 `createrepo_c` で repodata を生成する
RPM Repoに必要なファイル等を生成する。

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:9 \
  bash -lc '
    dnf install -y createrepo_c &&
    createrepo_c /work/rpm
  '
```

### 2.3 生成結果を確認する

```bash
tree
```

期待する構成例:

```text
.
├── hello-rpm
├── hello-rpm.spec
└── rpm
    ├── hello-rpm-0.1.0-1.el9.noarch.rpm
    └── repodata
        ├── ...-filelists.xml.gz
        ├── ...-primary.xml.gz
        ├── ...-other.xml.gz
        └── repomd.xml
```

`repodata/repomd.xml` が存在すれば、RPM リポジトリとして最低限必要な構成は揃っている。

---

## 3. GitHub Pages 用リポジトリに配置する

### 3.1 公開用リポジトリのローカル作業ディレクトリへ移動する

例:

```bash
cd ~/working/git/GitHub/rpm-repo
```

### 3.2 `rpm` ディレクトリをコピーする

```bash
cp -R ~/tmp/hello-rpm/rpm .
```

### 3.3 ディレクトリ構成を確認する

```bash
tree
```

期待する構成:

```text
.
└── rpm
    ├── hello-rpm-0.1.0-1.el9.noarch.rpm
    └── repodata
        └── repomd.xml
```

---

## 4. GitHub リポジトリに push する

### 4.1 Git 初期化する

```bash
git init
git branch -M main
```

### 4.2 コミットする

```bash
git add .
git commit -m "Initial RPM repository"
```

### 4.3 GitHub リポジトリと紐付ける

HTTPS ではなく SSH を使う構成にすると扱いやすい。
リポジトリーは公開するGitHubアカウント、プロジェクトに合わせて。
わたしのユーザー、このリポジトリーを指定する場合は次のように実施。

```bash
git remote add origin git@github.com:ytooyama/rpm-repo.git
```

すでに `origin` がある場合は、次で差し替える。

```bash
git remote set-url origin git@github.com:ytooyama/rpm-repo.git
```

### 4.4 push する

```bash
git push -u origin main
```

---

## 5. GitHub Pages を有効化する

GitHub の `ytooyama/rpm-repo` リポジトリ画面で、以下を設定する。

1. `Settings`
2. `Pages`
3. `Build and deployment`
   - Source: `Deploy from a branch`
   - Branch: `main`
   - Folder: `/ (root)`
4. `Save`

公開 URL の例:

```text
https://ytooyama.github.io/rpm-repo/
```

ただし、ルートには `index.html` がないため、トップ URL は 404 でも問題ない。
確認すべきなのは RPM リポジトリ本体の URL である。

---

## 6. GitHub Pages 上で公開確認する

以下の URL をブラウザで開く。`ユーザー名.github.io/github.io/rpm-repo/rpm/repodata/repomd.xml`を開くと、dnf/yumコマンドを実行してリポジトリーにアクセスしてrpmパッケージを取得する際に必要な`repomd.xml`の内容が表示されるはず。

```text
https://ytooyama.github.io/rpm-repo/rpm/repodata/repomd.xml
```

この XML が表示されれば、GitHub Pages 上で RPM リポジトリが正しく公開されている。

---

## 7. Linux ホストで利用設定する

### 7.1 `yum.repos.d` に repo 定義を作成する

```bash
sudo tee /etc/yum.repos.d/ytooyama-test.repo >/dev/null <<'EOF'
[ytooyama-test]
name=ytooyama test repo
baseurl=https://ytooyama.github.io/rpm-repo/rpm/
enabled=1
gpgcheck=0
EOF
```

### 7.2 メタデータキャッシュを更新する

```bash
sudo dnf clean all
sudo dnf makecache
```

### 7.3 利用可能なパッケージを確認する

```bash
sudo dnf list hello-rpm
```

期待する例:

```text
利用可能なパッケージ
hello-rpm.noarch   0.1.0-1.el9   ytooyama-test
```

---

## 8. Linux ホストでインストール確認する

### 8.1 インストールする

```bash
sudo dnf install -y hello-rpm
```

### 8.2 実行確認する

```bash
hello-rpm
```

期待する出力:

```text
Hello from hello-rpm
```

ここまで確認できれば、GitHub Pages 上の RPM リポジトリから配布・インストール・実行まで一通り成功している。

---

## 9. リポジトリの内容を変更した後の作業

RPM を追加、更新、削除したときは、単にファイルを置き換えるだけでは不十分である。
`repodata` を更新し、その内容を GitHub に push し直す必要がある。

---

## 10. RPM を追加または更新した場合

### 10.1 新しい RPM を `rpm/` ディレクトリへ配置する

例:

```bash
cp /path/to/new-package.rpm ~/working/git/GitHub/rpm-repo/rpm/
```

既存 RPM を更新する場合は、古い RPM を削除してから新しい RPM を配置する。

例:

```bash
rm -f ~/working/git/GitHub/rpm-repo/rpm/hello-rpm-0.1.0-1.el9.noarch.rpm
cp /path/to/hello-rpm-0.2.0-1.el9.noarch.rpm ~/working/git/GitHub/rpm-repo/rpm/
```

### 10.2 `repodata` を更新する

作業ディレクトリで次を実行する。

```bash
cd ~/working/git/GitHub/rpm-repo
```

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:9 \
  bash -lc '
    dnf install -y createrepo_c &&
    createrepo_c --update /work/rpm
  '
```

### 10.3 GitHub に反映する

```bash
git add .
git commit -m "Update RPM repository"
git push
```

### 10.4 公開確認する

ブラウザで以下を開き、`repomd.xml` が引き続き見えることを確認する。

```text
https://ytooyama.github.io/rpm-repo/rpm/repodata/repomd.xml
```

### 10.5 Linux 側でキャッシュを更新する

```bash
sudo dnf clean all
sudo dnf makecache
```

必要に応じて更新確認:

```bash
sudo dnf list hello-rpm
sudo dnf upgrade -y hello-rpm
```

---

## 11. RPM を削除した場合

RPM を削除した場合も、`repodata` を再生成または更新しなければならない。

### 11.1 不要な RPM を削除する

```bash
rm -f ~/working/git/GitHub/rpm-repo/rpm/old-package.rpm
```

### 11.2 `repodata` を更新する

```bash
cd ~/working/git/GitHub/rpm-repo
```

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:9 \
  bash -lc '
    dnf install -y createrepo_c &&
    createrepo_c --update /work/rpm
  '
```

必要に応じて、古い `repodata` をいったん消して完全再生成してもよい。

```bash
rm -rf rpm/repodata
```

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:9 \
  bash -lc '
    dnf install -y createrepo_c &&
    createrepo_c /work/rpm
  '
```

### 11.3 GitHub に反映する

```bash
git add .
git commit -m "Remove package from RPM repository"
git push
```

### 11.4 Linux 側でキャッシュを更新する

```bash
sudo dnf clean all
sudo dnf makecache
```

---

## 12. 日常運用時の基本手順

RPM リポジトリの内容を変更したときは、基本的に毎回以下の流れになる。

1. `rpm/` ディレクトリ内の RPM を追加・更新・削除する
2. `createrepo_c --update rpm/` を実行する
3. `git add .`
4. `git commit`
5. `git push`
6. 利用側ホストで `dnf clean all && dnf makecache` を実行する

---

## 13. 補足

### 13.1 GitHub Pages のルート URL が 404 でも問題ない

以下の URL が 404 でも問題ない。

```text
https://ytooyama.github.io/rpm-repo/
```

ルートに `index.html` がないだけであり、RPM リポジトリとしては正常である。
重要なのは、`rpm/repodata/repomd.xml` が見えることである。

### 13.2 今回は `gpgcheck=0` で動作確認している

今回の構成はテスト目的であり、repo 定義では次のようにしている。

```ini
gpgcheck=0
```

本番運用では以下を検討すること。

- RPM 署名
- リポジトリメタデータ署名
- 公開鍵配布
- `gpgcheck=1`

### 13.3 今回の `hello-rpm` は `noarch`

今回作成した RPM は `BuildArch: noarch` のため、CPU アーキテクチャ非依存である。
そのため、x86_64 / aarch64 の違いを気にせず配布テストに使いやすい。

---

## 14. まとめ

今回の手順で、以下を確認できた。

- Mac + Podman でテスト用 RPM を作成できる
- `createrepo_c` で RPM リポジトリを作成できる
- GitHub Pages で RPM リポジトリを公開できる
- Linux ホストから `dnf` で参照できる
- パッケージをインストールして実行確認できる

このため、GitHub Pages を使った最小構成の RPM 配布基盤としては成立している。
