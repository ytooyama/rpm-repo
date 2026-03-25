# GitHub Pages を使った RPM リポジトリ構築手順（EL9 / EL10対応版）

## 概要

この手順では以下を実施する。

1. テスト用 RPM パッケージを作成する
2. EL9 / EL10 用に RPM を分離する
3. `createrepo_c` で RPM リポジトリを作成する
4. GitHub Pages で公開する
5. Linux ホストから `dnf` で利用確認する
6. リポジトリ更新時の手順を整理する

## 前提

- GitHub アカウントを持っている
- GitHub 上に公開用リポジトリを作成済みである
  - 例: `ytooyama/rpm-repo`
- Podman が利用可能である
- Linux ホスト側で `dnf` が利用可能である
- テスト用 RPM は `noarch` を使用する
- GitHub への push は SSH を利用できる状態である

## 確認した環境

### パッケージ作成や GitHub 関連の作業をした環境

- macOS Tahoe 26.3.1
- Podman Desktop

### パッケージインストールをした環境

- Rocky Linux 9.7

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

ここはテスト用の内容である。すでに公開したい RPM パッケージがある場合は省略できる。

```bash
cat > hello-rpm <<'EOF'
#!/bin/bash
echo "Hello from hello-rpm"
EOF

chmod +x hello-rpm
```

### 1.3 SPEC ファイルを作成する

ここはテスト用の内容である。すでに公開したい RPM パッケージがある場合は省略できる。

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

うっかり防止のため、Podman コンテナで作業する。

ここはテスト用の内容である。すでに公開したい RPM パッケージがある場合は省略できる。

#### EL9 用 RPM を作成する

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

#### EL10 用 RPM を作成する

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:10 \
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

ここはテスト用の内容である。すでに公開したい RPM パッケージがある場合は省略できる。

```bash
ls -lh *.rpm
```

成功すると、例えば次のようなファイルができる。

```text
hello-rpm-0.1.0-1.el9.noarch.rpm
hello-rpm-0.1.0-1.el10.noarch.rpm
```

---

## 2. RPM リポジトリを作成する

ここから先は、GitHub Pages で RPM パッケージを公開したいときに実行する手順である。

### 2.1 RPM 配置用ディレクトリを作成する

EL9 / EL10 のように複数のディストリビューションを扱う場合は、RPM をディレクトリごとに分離する。

```bash
mkdir -p rpm/el9 rpm/el10

mv hello-rpm-*.el9*.rpm rpm/el9/
mv hello-rpm-*.el10*.rpm rpm/el10/
```

### 2.2 `createrepo_c` で repodata を生成する

RPM Repo に必要なファイルを生成する。  
`createrepo_c` はディレクトリごとに実行する。

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:9 \
  bash -lc '
    dnf install -y createrepo_c &&
    createrepo_c /work/rpm/el9 &&
    createrepo_c /work/rpm/el10
  '
```

### 2.3 生成結果を確認する

```bash
tree rpm
```

期待する構成例:

```text
rpm
├── el10
│   ├── hello-rpm-0.1.0-1.el10.noarch.rpm
│   └── repodata
│       ├── ...
│       └── repomd.xml
└── el9
    ├── hello-rpm-0.1.0-1.el9.noarch.rpm
    └── repodata
        ├── ...
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
tree rpm
```

期待する構成:

```text
rpm
├── el10
│   ├── hello-rpm-0.1.0-1.el10.noarch.rpm
│   └── repodata
│       ├── ...
│       └── repomd.xml
└── el9
    ├── hello-rpm-0.1.0-1.el9.noarch.rpm
    └── repodata
        ├── ...
        └── repomd.xml
```

---

## 4. GitHub リポジトリに push する

### 4.1 Git 初期化する

既存の GitHub リポジトリと紐付いたローカルリポジトリであれば不要。  
まだ Git 初期化していない場合のみ実行する。

```bash
git init
git branch -M main
```

### 4.2 コミットする

```bash
git add .
git commit -m "Update RPM repository"
```

### 4.3 GitHub リポジトリと紐付ける

HTTPS ではなく SSH を使う構成にすると扱いやすい。  
リポジトリは公開する GitHub アカウント、プロジェクトに合わせて設定する。  
`ytooyama/rpm-repo` を使う場合は次のように実施する。

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

以下の URL をブラウザで開く。  
`repomd.xml` の内容が表示されれば、GitHub Pages 上で RPM リポジトリが正しく公開されている。

### EL9

```text
https://ytooyama.github.io/rpm-repo/rpm/el9/repodata/repomd.xml
```

### EL10

```text
https://ytooyama.github.io/rpm-repo/rpm/el10/repodata/repomd.xml
```

---

## 7. Linux ホストで利用設定する

### 7.1 `yum.repos.d` に repo 定義を作成する

EL9 ホストでの例:

```bash
sudo tee /etc/yum.repos.d/ytooyama-test.repo >/dev/null <<'EOF'
[ytooyama-test]
name=ytooyama test repo
baseurl=https://ytooyama.github.io/rpm-repo/rpm/el9/
enabled=1
gpgcheck=0
EOF
```

`$releasever` を使って自動切り替えしたい場合は、次のようにも書ける。

```bash
sudo tee /etc/yum.repos.d/ytooyama-test.repo >/dev/null <<'EOF'
[ytooyama-test]
name=ytooyama test repo
baseurl=https://ytooyama.github.io/rpm-repo/rpm/el$releasever/
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

### 10.1 新しい RPM を対応するディレクトリへ配置する

例:

```bash
cp /path/to/new-package-1.0-1.el9.noarch.rpm ~/working/git/GitHub/rpm-repo/rpm/el9/
cp /path/to/new-package-1.0-1.el10.noarch.rpm ~/working/git/GitHub/rpm-repo/rpm/el10/
```

既存 RPM を更新する場合は、古い RPM を削除してから新しい RPM を配置する。

例:

```bash
rm -f ~/working/git/GitHub/rpm-repo/rpm/el9/hello-rpm-0.1.0-1.el9.noarch.rpm
cp /path/to/hello-rpm-0.2.0-1.el9.noarch.rpm ~/working/git/GitHub/rpm-repo/rpm/el9/

rm -f ~/working/git/GitHub/rpm-repo/rpm/el10/hello-rpm-0.1.0-1.el10.noarch.rpm
cp /path/to/hello-rpm-0.2.0-1.el10.noarch.rpm ~/working/git/GitHub/rpm-repo/rpm/el10/
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
    createrepo_c --update /work/rpm/el9 &&
    createrepo_c --update /work/rpm/el10
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
https://ytooyama.github.io/rpm-repo/rpm/el9/repodata/repomd.xml
https://ytooyama.github.io/rpm-repo/rpm/el10/repodata/repomd.xml
```

### 10.5 Linux 側でキャッシュを更新する

```bash
sudo dnf clean all
sudo dnf makecache
```

必要に応じて更新確認する。

```bash
sudo dnf list hello-rpm
sudo dnf upgrade -y hello-rpm
```

---

## 11. RPM を削除した場合

RPM を削除した場合も、`repodata` を再生成または更新しなければならない。

### 11.1 不要な RPM を削除する

```bash
rm -f ~/working/git/GitHub/rpm-repo/rpm/el9/old-package.rpm
rm -f ~/working/git/GitHub/rpm-repo/rpm/el10/old-package.rpm
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
    createrepo_c --update /work/rpm/el9 &&
    createrepo_c --update /work/rpm/el10
  '
```

必要に応じて、古い `repodata` をいったん消して完全再生成してもよい。

```bash
rm -rf rpm/el9/repodata
rm -rf rpm/el10/repodata
```

```bash
podman run --rm -it \
  -v "$PWD":/work \
  -w /work \
  rockylinux/rockylinux:9 \
  bash -lc '
    dnf install -y createrepo_c &&
    createrepo_c /work/rpm/el9 &&
    createrepo_c /work/rpm/el10
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

1. `rpm/el9/` と `rpm/el10/` 内の RPM を追加・更新・削除する
2. `createrepo_c --update rpm/el9` を実行する
3. `createrepo_c --update rpm/el10` を実行する
4. `git add .`
5. `git commit`
6. `git push`
7. 利用側ホストで `dnf clean all && dnf makecache` を実行する

---

## 13. 補足

### 13.1 GitHub Pages のルート URL が 404 でも問題ない

以下の URL が 404 でも問題ない。

```text
https://ytooyama.github.io/rpm-repo/
```

ルートに `index.html` がないだけであり、RPM リポジトリとしては正常である。  
重要なのは、各ディレクトリ配下の `repodata/repomd.xml` が見えることである。

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

### 13.4 `createrepo_c` の実行コンテナは Rocky Linux 9 で問題ない

`createrepo_c` は RPM ヘッダを読み取ってメタデータを生成するツールであり、  
EL9 用 repo と EL10 用 repo の両方に対して Rocky Linux 9 コンテナから実行して問題ない。  
重要なのは、ディレクトリを分けて `createrepo_c` を実行することである。

---

## 14. まとめ

今回の手順で、以下を確認できた。

- Mac + Podman でテスト用 RPM を作成できる
- EL9 / EL10 ごとにディレクトリを分離して RPM リポジトリを作成できる
- `createrepo_c` で RPM リポジトリを作成できる
- GitHub Pages で RPM リポジトリを公開できる
- Linux ホストから `dnf` で参照できる
- パッケージをインストールして実行確認できる

このため、GitHub Pages を使った最小構成の RPM 配布基盤としては成立している。
