# 本日のCI/CDで出会ったエラー・課題

## 背景

既存の `task-app` に対して、GitHub Actions 用の `workflows` の `yml` ファイルを作成し、CI/CD環境を構築した。

---

# 発生したエラー・課題

## 1. View の Build エラー

### 内容

Laravel の View 関連で build エラーが発生。

### 原因

フロントエンドのビルド処理（Vite）が事前に実行されておらず、必要なアセットが生成されていなかった。

---

## 2. DB接続エラー

### 内容

```bash
SQLSTATE[HY000] [1045] Access denied for user 'sail'@'172.18.0.1'
(using password: YES)
(Connection: mysql, Host: 127.0.0.1, Port: 3306, Database: testing)
```

### 原因

GitHub Actions 上で `.env` が正しく設定されておらず、
Laravel Sail 用の接続情報を参照していた。

### 解決方法

GitHub Actions 内で DB 接続情報を明示的に `.env` に追記した。

```yaml
echo "DB_CONNECTION=mysql" >> .env
echo "DB_HOST=127.0.0.1" >> .env
echo "DB_PORT=3306" >> .env
echo "DB_DATABASE=laravel_test" >> .env
echo "DB_USERNAME=root" >> .env
echo "DB_PASSWORD=root" >> .env
```

---

## 3. APP_KEY 設定エラー

### 内容

```bash
No application encryption key has been specified.
(View: /home/runner/work/task-app/task-app/storage/framework/views/xxxx.blade.php)
```

### 原因

CI環境で `APP_KEY` が生成されていなかった。

### 解決方法

CI実行時に以下を実行して Laravel のキーを生成。

```bash
php artisan key:generate
```

---

## 4. テストエラー

### 内容

```bash
Failed asserting that two strings are identical
```

```bash
Route [tasks] not defined.
```

など、Feature Test / Unit Test 関連のエラーが発生。

### 原因

* 期待値と実際のレスポンスが一致していない
* Route 名の設定漏れ
* テスト用データやリダイレクト先の不一致

### 対応

* Route 定義を確認
* `web.php` の route 名を修正
* テストコードの期待値を見直し

---

## 5. Vite 関連エラー

### 内容

Vite を使用していたため、フロントエンド build エラーが発生。

### 原因

CI環境で npm パッケージのインストールや build が未実行だった。

### 解決方法

GitHub Actions 内で事前セットアップを追加。

```yaml
- name: Install dependencies
  run: composer install --no-interaction --prefer-dist

- name: Install npm dependencies
  run: npm install

- name: Build frontend assets
  run: npm run build
```

---

# 学んだこと

* GitHub Actions では `.env` を明示的に設定する必要がある
* Laravel の `APP_KEY` は CI 環境でも必須
* MySQL 接続先は Docker 環境と CI 環境で異なる場合がある
* Vite 使用時は `npm install` と `npm run build` が必要
* CI では「ローカルでは動くが CI では失敗する」ケースが多い
* Route 名や期待値のズレでテストが失敗することがある

---

# 今後の改善点

* `.env.testing` を作成して CI 用設定を分離する
* reusable workflow 化して workflow を整理する
* test / build / deploy をブランチごとに分離する
* PR 時のみ test 実行、本番 merge 時のみ deploy 実行などルールを明確化する
