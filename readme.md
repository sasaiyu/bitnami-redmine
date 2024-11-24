# bitnami-redmine ＆redmine_persist_wfmt

[bitnami/redmine](https://hub.docker.com/r/bitnami/redmine)のコンテナを作成して、プラグイン[redmine_persist_wfmt](https://github.com/pinzolo/redmine_persist_wfmt)を追加する。

なお、[redmine_persist_wfmt](https://github.com/pinzolo/redmine_persist_wfmt)の readme 通りにプラグインを入れようとすると[エラー](#原因について)になるのでプラグインを修正した。

## TL;DR

```bash
# Redmineの環境は構築されている前提
# 未構築の場合は Docker イメージの章を参照

# プラグインをクーロンしてGemfileを更新
cd /opt/bitnami/redmine
bundle config unset deployment
git clone https://github.com/pinzolo/redmine_persist_wfmt.git plugins/redmine_persist_wfmt
bundle install

# 本番環境のGemfileを更新
bundle config set --local deployment 'true'
bundle config set --local without 'test development'
bundle install

# pwfmt.rbをpwfmt.rb.bakに複製し、12行目以降を削除
cd /opt/bitnami/redmine/plugins/redmine_persist_wfmt/lib
sed -i.bak '12,$d' pwfmt.rb

# 動作確認
cd /opt/bitnami/redmine && bundle exec rake zeitwerk:check RAILS_ENV=production
# エラーにならず、OKが返ってくること
# Hold on, I am eager loading the application.
# All is good!

# サーバを再起動して完了
```

以降は TL;DR に至るまでの過程を記録するためのものであるため読み飛ばしてよい。

## [Docker イメージ](https://hub.docker.com/layers/bitnami/redmine/5.0.5-debian-11-r15/images/sha256-556ece25bf13133f54ffb55b782d4977daadc4b1832834da3e41b4e7d3ae3537?context=explore)

### [bitnami/redmine](https://hub.docker.com/r/bitnami/redmine) の [compose.yaml](#bitnamiredmine-の-composeyaml-のダウンロード) の変更点

Docker イメージのバージョンを[5.0.5-debian-11-r15](https://hub.docker.com/layers/bitnami/redmine/5.0.5-debian-11-r15/images/sha256-556ece25bf13133f54ffb55b782d4977daadc4b1832834da3e41b4e7d3ae3537?context=explore)に変更

```yaml
services:
  redmine:
    image: docker.io/bitnami/redmine:5.0.5-debian-11-r15
```

### コンテナの起動と削除

```bash
# 起動
docker compose up -d

# ボリュームごと削除
docker compose down --volumes --remove-orphans
```

### [Redmine へのプラグインの追加](https://docs.bitnami.com/general/apps/redmine/configuration/install-plugins/)

Markdown と textile を両方利用できるように、プラグイン [redmine_persist_wfmt](https://github.com/pinzolo/redmine_persist_wfmt) を追加する。なお、readme だと対応する Redmine のバージョンが 4.0.x となっているが、[issue #22](https://github.com/pinzolo/redmine_persist_wfmt/issues/22)で 5.1 に対応してる様子（[master branch にも merge されてる](https://github.com/pinzolo/redmine_persist_wfmt/pull/23)）

なお、[redmine_persist_wfmt](https://github.com/pinzolo/redmine_persist_wfmt) の手順と異なり、deployment モードをオフにして Gemifile を編集できるようにする。これをしないと Gemfile.lock と Gemfile の不整合により、そもそもコンテナが起動できなくなる。[詳しくはこちら](https://kajindowsxp.com/unable-bundle-install/)。

```bash
cd /opt/bitnami/redmine && ls
# CONTRIBUTING.md  bin        licenses                 test
# Gemfile          config     log                      tmp
# Gemfile.lock     config.ru  package.json             vendor
# README.rdoc      db         passenger.3000.pid       yarn.lock
# Rakefile         extra      passenger.3000.pid.lock
# app              files      plugins
# appveyor.yml     lib        public

# Gemfileを更新するために開発環境に切り替える
bundle config unset deployment

git clone https://github.com/pinzolo/redmine_persist_wfmt.git plugins/redmine_persist_wfmt

bundle install
# Bundle complete! 46 Gemfile dependencies, 77 gems now # installed.
# Gems in the groups 'test' and 'development' were not # installed.
# Bundled gems are installed into `./vendor/bundle`

# 本番環境にもインストール
bundle config set --local deployment 'true'
bundle config set --local without 'test development'

bundle install

bundle exec rake redmine:plugins:migrate NAME=redmine_persist_wfmt RAILS_ENV=production
# rake aborted!
# ActiveSupport::Concern::MultipleIncludedBlocks: Cannot define multiple 'included' # blocks for a Concern
# /opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/# active_support/concern.rb:160:in `included'
# /bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt/patches/controller/application_controller_patch.rb:10:in `<module:ApplicationControllerPatch>'
```

**しかし、プラグインマイグレーションでエラー[ (スタックトレースはこちら) ](#プラグインマイグレーションでのエラー)がでてしまう・・・**

### エラーの内容

`extend ActiveSupport::Concern` は[クラスメソッドを定義](https://zenn.dev/ebina_shohei/articles/d2a6526533a120)するためのものだが、同じモジュール内で複数の `included` が定義されている、正確には二重に呼ばれているのが問題。

```ruby
# plugins/redmine_persist_wfmt/lib/pwfmt/patches/controller/application_controller_patch.rb
module Pwfmt
  module Patches
    module Controller
      module ApplicationControllerPatch
        extend ActiveSupport::Concern

        # ここでエラーが発生
        included do
          before_action :store_pwfmt_params
          after_action :clear_pwfmt_context
        end
```

### 原因について

#### Zeitwerk（ツァイトベルク）によるオートロード

[Redmine 5 から rails 6 の Zeitwerk モードが有効](https://zenn.dev/onozaty/articles/redmine5-plugin-migration) になっているが、Zeitwerk により動作する[オートロード](https://railsguides.jp/autoloading_and_reloading_constants.html#:~:text=%E8%87%AA%E5%8B%95%E8%AA%AD%E3%81%BF%E8%BE%BC%E3%81%BF%E3%83%91%E3%82%B9%EF%BC%88autoload%20path,%E3%81%82%E3%82%8B%20Object%20%E3%82%92%E8%A1%A8%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)
と プラグイン内での `require` の指定により二重に `ActiveSupport::Concern` が読み込みされてるのが[原因](https://stackoverflow.com/questions/24244519/cannot-define-multiple-included-blocks-for-a-concern-activesupportconcern?noredirect=1&lq=1)。

```
cd /opt/bitnami/redmine/ && bin/rails --version
# Rails 6.1.7.2
```

そのため、プラグインの修正が必要となる。

[Redmine 5.0 で古いプラグインを動作させる方法](https://www.farend.co.jp/blog/2022/03/redmine-plugins-zeitwerk/)

#### プラグインの修正

`redmine_persist_wfmt/lib/pwfmt.rb` の 12 行目以降の `require` を[削除](https://techracho.bpsinc.jp/hachi8833/2024_03_14/139702)すればよい。

```bash
cd /opt/bitnami/redmine/plugins/redmine_persist_wfmt/lib
sed -i.bak '12,$d' pwfmt.rb
diff pwfmt.rb pwfmt.rb.bak
# 11a12,33
# > require_relative 'pwfmt/patches/controller/application_controller_patch'
# > require_relative 'pwfmt/patches/controller/documents_controller_patch'
# > require_relative 'pwfmt/patches/controller/issues_controller_patch'
# > require_relative 'pwfmt/patches/controller/journals_controller_patch'
# > require_relative 'pwfmt/patches/controller/messages_controller_patch'
# > require_relative 'pwfmt/patches/controller/news_controller_patch'
# > require_relative 'pwfmt/patches/controller/previews_controller_patch'
# > require_relative 'pwfmt/patches/controller/projects_controller_patch'
# > require_relative 'pwfmt/patches/controller/settings_controller_patch'
# > require_relative 'pwfmt/patches/controller/welcome_controller_patch'
# > require_relative 'pwfmt/patches/controller/wiki_controller_patch'
# >
# > require_relative 'pwfmt/patches/model/comment_patch'
# > require_relative 'pwfmt/patches/model/document_patch'
# > require_relative 'pwfmt/patches/model/issue_patch'
# > require_relative 'pwfmt/patches/model/journal_patch'
# > require_relative 'pwfmt/patches/model/message_patch'
# > require_relative 'pwfmt/patches/model/news_patch'
# > require_relative 'pwfmt/patches/model/project_patch'
# > require_relative 'pwfmt/patches/model/setting_patch'
# > require_relative 'pwfmt/patches/model/wiki_content_patch'
# > require_relative 'pwfmt/patches/model/wiki_content_version_patch'
```

#### 削除後の動作確認

Zeitwerk にプラグインが対応しているか確認

```bash
cd /opt/bitnami/redmine && bundle exec rake zeitwerk:check RAILS_ENV=productions
# Hold on, I am eager loading the application.
# All is good!
```

上記が正常であれば問題ないが、一応[オートロードが見ているパス](https://blog.cloud-acct.com/posts/column-rails6-zeitwerk-autoload/)やモジュールを確認する

```
cd /opt/bitnami/redmine && ./bin/rails console -e production

# コンソールが起動したら以下のコマンドをそれぞれ入力
puts ActiveSupport::Dependencies.autoload_paths
# ファイルパスが表示されればよい

puts %i[
  ApplicationControllerPatch
  DocumentsControllerPatch
  IssuesControllerPatch
  JournalsControllerPatch
  MessagesControllerPatch
  NewsControllerPatch
  PreviewsControllerPatch
  ProjectsControllerPatch
  SettingsControllerPatch
  WelcomeControllerPatch
  WikiControllerPatch
].all? { |_| Pwfmt::Patches::Controller.const_defined?(_) }

puts %i[
  CommentPatch
  DocumentPatch
  IssuePatch
  JournalPatch
  MessagePatch
  NewsPatch
  ProjectPatch
  SettingPatch
  WikiContentPatch
  WikiContentVersionPatch
].all? { |_| Pwfmt::Patches::Model.const_defined?(_) }
# true が表示されればモジュールはロード済み
```

なお、[development](https://qiita.com/fumi19/items/9c0d3544ab4b2c78023f) 環境だと `listen` のバージョン問題で[エラー](#development-環境でのエラー)になり起動しない。

## Appendix

### [bitnami/redmine](https://hub.docker.com/r/bitnami/redmine) の compose.yaml のダウンロード

```bash
curl -sSL https://raw.githubusercontent.com/bitnami/containers/main/bitnami/redmine/docker-compose.yml > docker-compose.yml
```

### プラグインマイグレーションでのエラー

```bash
bundle exec rake redmine:plugins:migrate NAME=redmine_persist_wfmt RAILS_ENV=production
#　rake aborted!
#　ActiveSupport::Concern::MultipleIncludedBlocks: Cannot define multiple 'included' blocks for a Concern
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/concern.rb:160:in `included'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt/patches/controller/application_controller_patch.rb:10:in `<module:ApplicationControllerPatch>'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt/patches/controller/application_controller_patch.rb:7:in `<module:Controller>'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt/patches/controller/application_controller_patch.rb:3:in `<module:Patches>'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt/patches/controller/application_controller_patch.rb:2:in `<module:Pwfmt>'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt/patches/controller/application_controller_patch.rb:1:in `<top (required)>'
#　/opt/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt.rb:12:in `require_relative'
#　/opt/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt.rb:12:in `<top (required)>'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/zeitwerk-2.6.7/lib/zeitwerk/kernel.rb:30:in `require'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt/context.rb:1:in `<top (required)>'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt.rb:1:in `require_relative'
#　/bitnami/redmine/plugins/redmine_persist_wfmt/lib/pwfmt.rb:1:in `<top (required)>'
#　/opt/bitnami/redmine/plugins/redmine_persist_wfmt/init.rb:10:in `require_relative'
#　/opt/bitnami/redmine/plugins/redmine_persist_wfmt/init.rb:10:in `<top (required)>'
#　/opt/bitnami/redmine/lib/redmine/plugin_loader.rb:31:in `load'
#　/opt/bitnami/redmine/lib/redmine/plugin_loader.rb:31:in `run_initializer'
#　/opt/bitnami/redmine/lib/redmine/plugin_loader.rb:108:in `each'
#　/opt/bitnami/redmine/lib/redmine/plugin_loader.rb:108:in `block in load'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:427:in `instance_exec'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:427:in `block in make_lambda'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:198:in `block (2 levels) in halting'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:604:in `block (2 levels) in default_terminator'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:603:in `catch'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:603:in `block in default_terminator'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:199:in `block in halting'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:512:in `block in invoke_before'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:512:in `each'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:512:in `invoke_before'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/callbacks.rb:105:in `run_callbacks'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/reloader.rb:88:in `prepare!'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/application/finisher.rb:124:in `block in <module:Finisher>'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:32:in `instance_exec'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:32:in `run'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:61:in `block in run_initializers'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:60:in `run_initializers'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/application.rb:391:in `initialize!'
#　/opt/bitnami/redmine/config/environment.rb:16:in `<top (required)>'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/zeitwerk-2.6.7/lib/zeitwerk/kernel.rb:38:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `block in require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:299:in `load_dependency'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/application.rb:367:in `require_environment!'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/application.rb:533:in `block in run_tasks_blocks'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/rake-13.0.6/exe/rake:27:in `<top (required)>'
#　/opt/bitnami/ruby/bin/bundle:25:in `load'
#　/opt/bitnami/ruby/bin/bundle:25:in `<main>'
#　Tasks: TOP => redmine:plugins:migrate => environment
#　(See full trace by running task with --trace)
```

### Development 環境でのエラー

`bundle install` で `listen` を[インストールすると解決する](https://www.tmp1024.com/rails-task-error-missing-listen/#google_vignette)とあったが効果なし・・・

[redmine は develop では動かない](https://github.com/docker-library/redmine/issues/298)可能性があり、恐らく[今も治っていない](https://www.redmine.org/issues/37048)

```bash
bundle exec rake zeitwerk:check
#　rake aborted!
#　LoadError: cannot load such file -- listen
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/zeitwerk-2.6.7/lib/zeitwerk/kernel.rb:38:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `block in require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:299:in `load_dependency'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/evented_file_update_checker.rb:6:in `<top (required)>'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/zeitwerk-2.6.7/lib/zeitwerk/kernel.rb:38:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `block in require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:299:in `load_dependency'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `require'
#　/opt/bitnami/redmine/config/environments/development.rb:58:in `block in <top (required)>'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/railtie.rb:234:in `instance_eval'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/railtie.rb:234:in `configure'
#　/opt/bitnami/redmine/config/environments/development.rb:5:in `<top (required)>'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/zeitwerk-2.6.7/lib/zeitwerk/kernel.rb:38:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `block in require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:299:in `load_dependency'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/engine.rb:571:in `block (2 levels) in <class:Engine>'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/engine.rb:570:in `each'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/engine.rb:570:in `block in <class:Engine>'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:32:in `instance_exec'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:32:in `run'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:61:in `block in run_initializers'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:50:in `each'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:50:in `tsort_each_child'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/initializable.rb:60:in `run_initializers'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/application.rb:391:in `initialize!'
#　/opt/bitnami/redmine/config/environment.rb:16:in `<top (required)>'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　<internal:/opt/bitnami/ruby/lib/ruby/site_ruby/3.0.0/rubygems/core_ext/kernel_require.rb>:37:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/zeitwerk-2.6.7/lib/zeitwerk/kernel.rb:38:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `block in require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:299:in `load_dependency'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/activesupport-6.1.7.2/lib/active_support/dependencies.rb:332:in `require'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/application.rb:367:in `require_environment!'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/railties-6.1.7.2/lib/rails/application.rb:533:in `block in run_tasks_blocks'
#　/opt/bitnami/redmine/vendor/bundle/ruby/3.0.0/gems/rake-13.0.6/exe/rake:27:in `<top (required)>'
#　/opt/bitnami/ruby/bin/bundle:25:in `load'
#　/opt/bitnami/ruby/bin/bundle:25:in `<main>'
#　Tasks: TOP => zeitwerk:check => environment
#　(See full trace by running task with --trace)
```

### [DB](https://redmine.jp/guide/RedmineInstall/#step-3-) について

起動時に mariaDB に`pwfmt_formats` テーブルが存在せずエラーになる場合がある。`mysql -u bn_redmine` または `mysql -u bn_redmine -p` （パスワード付きの場合は設定ファイル `config/database.yml` を参照）でログインし、テーブルを作成すればよい。

```SQL
MariaDB [(none)]> USE bitnami_redmine;

MariaDB [bitnami_redmine]> show create table pwfmt_formats\G;
--　*************************** 1. row ***************************
--　       Table: pwfmt_formats
--　Create Table: CREATE TABLE `pwfmt_formats` (
--　  `id` bigint(20) NOT NULL AUTO_INCREMENT,
--　  `target_id` int(11) NOT NULL,
--　  `field` varchar(255) NOT NULL,
--　  `format` varchar(255) NOT NULL,
--　  `created_at` datetime NOT NULL,
--　  `updated_at` datetime NOT NULL,
--　  PRIMARY KEY (`id`),
--　  UNIQUE KEY `pwfmt_formats_uniq_idx` (`target_id`,`field`)
--　) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
--　1 row in set (0.003 sec)
```

### NG だったが関連しそうな内容

`run: bundle install --path vendor/cache` で[治るかもとあった](https://qiita.com/k-mashi/items/fdf7c962dbf95e0a55fa)が、効果なし・・・

`bundle install --path` を指定すると `bundler` の[インストールパスが変わるため注意](https://qiita.com/Nedward/items/55b8dac4001a514b9bcc)

```
bitnami@ip-10-0-1-183:/opt/bitnami/redmine$ bundle config
Settings are listed in order of priority. The top value will be used.
bin
Set for your local app (/opt/bitnami/redmine/.bundle/config) : "bin"

gemfile
Set via BUNDLE_GEMFILE: "/opt/bitnami/redmine/Gemfile"

path
Set for your local app (/opt/bitnami/redmine/.bundle/config) : "vendor/bundle"

without
Set for your local app (/opt/bitnami/redmine/.bundle/config) : [: test, : development]
```
