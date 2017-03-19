[トップページに戻る](../README.md) | [前回: 仮想マシンについて](201703_2nd-study.md)

# TBD

## 初心に戻る

アプリを作成したときのスタート地点、つまり「開始」タブのコンテンツに戻ってみましょう。Bluemix が勧めてくるのは、以下のように GitHub のサンプルをローカルにもってくることです。

```sh
git clone https://github.com/IBM-Bluemix/get-started-node
```

わざわざローカルに clone しなくても、GitHub サイトで内容を確認することができます。

[https://github.com/IBM-Bluemix/get-started-node](https://github.com/IBM-Bluemix/get-started-node)

## 仮想マシンの基本設定

昨日、仮想マシンのメモリ容量を変更した時に、以下のような警告が表示されました。

![Web 警告ポップアップ](i/201703_3rd-scripts_01.png)

仮想マシンの設定は [manifest.yml](https://github.com/IBM-Bluemix/get-started-node/blob/master/manifest.yml) ファイルだそうです。Bluemix コンソール上でメモリ容量を修正しても、cf push してDroplet(仮想マシンの原型)を更新したら、このファイルの値で上書きされちゃうよ、ということですね。

```
---
applications:
 - name: GetStartedNode
   random-route: true
   memory: 256M
```

確かにメモリ容量が 256MB に設定されています。

random-route は「アプリケーションにランダムな経路を割り当てる」って説明がありますが、これは xxx.myblluemix.net というアクセス用の URL の xxx を自動で決めてくれる、のでしょうかね。

## アプリ起動の様子

OS 自体の起動は /sbin/init からで、upstart です。しかし /etc/init/\*.conf は多くあって、いちいち見てたら大変です。

![cf ssh コンソール](i/201703_3rd-scripts_02.png)

まあこのあたりは Ubuntu の基本だしいいよね、ということで ps aux で確認できる app/.app-management/scripts/start は… [初日](201703_1st-step.md#さてどうなっているのか) で軽く見ましたね。

もう少し深く見るため、呼ばれている app/.app-management/utils/handler_utils.sh を見てみましょう。

```sh
#!/usr/bin/env bash
# IBM WebSphere Application Server Liberty Buildpack
# Copyright 2016 the original author or authors. (略)

function handler_port() {
  handler=$1
  param=$2
  default=$3
  INSTALL_DIR=$(cd `dirname ${BASH_SOURCE[0]}`/../.. && pwd)
  ruby <<-EORUBY
APP_MGMT_DIR = File.join('$INSTALL_DIR', '.app-management').freeze
\$LOAD_PATH.unshift File.expand_path(APP_MGMT_DIR, __FILE__)
require 'utils/handler_utils.rb'
config = Utils::HandlerUtils.get_configuration('$handler')
puts config['$param'] || $default
EORUBY
}

function enabled_handlers() {
  INSTALL_DIR=$(cd `dirname ${BASH_SOURCE[0]}`/../.. && pwd)
  ruby <<EORUBY
APP_MGMT_DIR = File.join('$INSTALL_DIR', '.app-management').freeze
\$LOAD_PATH.unshift File.expand_path(APP_MGMT_DIR, __FILE__)
require 'utils/enabled_handlers'
enabled = Utils.get_enabled_handlers
puts enabled.join("\n") unless enabled.nil?
EORUBY
}
```

これ、以下の app/.app-management/utils/ にある handler_utils.rb と、

```rb
# Encoding: utf-8

require 'yaml'
require 'utils/simple_logger'

module Utils
  class HandlerUtils
    def self.get_configuration(handler_name)
      var_name      = environment_variable_name(handler_name)
      user_provided = ENV[var_name]
      if user_provided
        begin
          user_provided_value = YAML.load(user_provided)
          return user_provided_value if user_provided_value.is_a?(Hash)
          SimpleLogger.error("Configuration value in environment variable #{var_name} is not valid: #{user_provided_value}")
        rescue Psych::SyntaxError => ex
          SimpleLogger.error("Configuration value in environment variable #{var_name} has invalid syntax: #{ex}")
        end
      end
      {}
    end

    ENVIRONMENT_VARIABLE_PATTERN = 'BLUEMIX_APP_MGMT_'.freeze

    def self.environment_variable_name(handler_name)
      ENVIRONMENT_VARIABLE_PATTERN + handler_name.upcase
    end

    private_constant :ENVIRONMENT_VARIABLE_PATTERN
    private_class_method :environment_variable_name
  end
end
```

enabled_handlers.rb への sh用ラッパーって感じですね。

```ruby
#!/usr/bin/env ruby
# IBM SDK for Node.js Buildpack (略)
require 'utils/simple_logger'
module Utils

  # Returns the list of enabled handlers set in the environment
  def Utils.get_enabled_handlers
    if ENV['ENABLE_BLUEMIX_DEV_MODE'] == 'TRUE' || ENV['ENABLE_BLUEMIX_DEV_MODE'] == 'true'
      %w{devconsole inspector shell}
    elsif !ENV['BLUEMIX_APP_MGMT_ENABLE'].nil?
      ENV['BLUEMIX_APP_MGMT_ENABLE'].downcase.split('+').map(&:strip)
    else
      nil
    end
  end
end
```

## TBD



cf ssh "Node.js Test" -c "cat app/vendor/initial_startup.rb"

```rb
#!/usr/bin/env ruby
# IBM SDK for Node.js Buildpack
# Copyright 2014 the original author or authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# points to /home/vcap/app
app_dir = File.expand_path('..', File.dirname(__FILE__))

app_mgmt_dir = File.join(app_dir, '.app-management')

$LOAD_PATH.unshift app_mgmt_dir

require 'json'
require 'utils/enabled_handlers'
require 'utils/handlers'
require 'utils/simple_logger'

def start_runtime(app_dir)
  exec(".app-management/scripts/start #{ENV['PORT']}", chdir: app_dir)
end

def start_proxy(app_dir)
  exec('.app-management/bin/proxyAgent', chdir: app_dir)
end

def get_environment(app_mgmt_dir, app_dir)
  env = {}
  env['BOOT_SCRIPT'] = ENV['BOOT_SCRIPT']
  env['BLUEMIX_DEV_CONSOLE_HIDE'] = '["stop"]'
  env['BLUEMIX_DEV_CONSOLE_START_TIMEOUT'] = '500'
  if system "bash -c \"source #{app_mgmt_dir}/utils/node_utils.sh && inspector_builtin #{app_dir}/vendor/node\""
    env['BLUEMIX_DEV_CONSOLE_TOOLS'] = '[ {"name":"shell", "label":"Shell"} ]'
  else
    env['BLUEMIX_DEV_CONSOLE_TOOLS'] = '[ {"name":"shell", "label":"Shell"}, {"name": "inspector", "label": "Debugger"} ]'
  end
  env
end

def run(app_dir, env, handlers, background)
  if handlers.length != 0
    command = handlers.map { | handler | handler.start_script }.join(' ; ')
    command = "( #{command} ) &" if background
    system(env, "#{command}", chdir: app_dir)
  end
end

def run_handlers(app_mgmt_dir, app_dir, handlers, valid_handlers, invalid_handlers)
  Utils::SimpleLogger.warning("Ignoring unrecognized app management utilities: #{invalid_handlers.join(', ')}") unless invalid_handlers.empty?
  Utils::SimpleLogger.info("Activating app management utilities: #{valid_handlers.join(', ')}")

  # get environment for handlers
  env = get_environment(app_mgmt_dir, app_dir)

  # sort handlers for sync and async execution
  sync_handlers, async_handlers = handlers.executions(valid_handlers)

  # execute sync handlers
  run(app_dir, env, sync_handlers, false)

  # execute async handlers
  run(app_dir, env, async_handlers, true)
end

def write_json(file, key, value)
  hash = JSON.parse(File.read(file))
  hash[key] = value
  File.open(file,"w") do |f|
    f.write(hash.to_json)
  end
end

handler_list = Utils.get_enabled_handlers

if handler_list.nil? || handler_list.empty?
  # No handlers are specified. Start the runtime normally.
  start_runtime(app_dir)
else
  handlers_dir = File.join(app_mgmt_dir, 'handlers')

  handlers = Utils::Handlers.new(handlers_dir)

  # validate headers
  valid_handlers, invalid_handlers = handlers.validate(handler_list)

  # check if proxy agent is required
  proxy_required = handlers.proxy_required?(valid_handlers)

  if proxy_required
    # check instance index
    index = JSON.parse(ENV['VCAP_APPLICATION'])['instance_index']
    if index != 0
      # Start the runtime normally. Only allow dev mode on index 0
      start_runtime(app_dir)
    else
      # Run handlers
      run_handlers(app_mgmt_dir, app_dir, handlers, valid_handlers, invalid_handlers)

      # Start proxy
      write_json(File.join(app_mgmt_dir, 'app_mgmt_info.json'), 'proxy_enabled', 'true')
      start_proxy(app_dir)
    end
  else
    # Run handlers
    run_handlers(app_mgmt_dir, app_dir, handlers, valid_handlers, invalid_handlers)

    # Start runtime
    start_runtime(app_dir)
  end
end
```


cf ssh "Node.js Test" -c "cat staging_info.yml"
```js
{
  "detected_buildpack":
    "SDK for Node.js(TM) (ibm-node.js-4.7.2, buildpack-v3.10-20170119-1146)",
  "start_command":"./vendor/initial_startup.rb"
}
```
