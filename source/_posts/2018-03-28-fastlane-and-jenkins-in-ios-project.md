---
title: iOS项目的fastlane/jenkins实践
date: 2018-03-28 13:47:47
tags: iOS
---

> 作者: 郑佳宁
> 原文: [https://www.jianshu.com/p/eafa2fa37c1b](https://www.jianshu.com/p/eafa2fa37c1b)

## 第一部分：TestFlight lane

**用途：**内测、公测、正式发布

**证书类型：**Production

**provisionning profile类型：**Distribution - app store

### fastlane构建过程：

1. 安装依赖
1. 运行测试
1. 更新 build number
1. 打包
1. 上传到testflight
1. 从iTunes Connect下载dSYMs、并上传至Crashlytics

### 每一步用到的fastlane命令对应如下：

``` ruby
cocoapods

scan

# 更新 build number: 
get_version_number
# （从testflight上读取最新的build number，并加 1，为新的build number）
latest_testflight_build_number
# （CFBundleVersion的值）
update_info_plist

gym

# （同 pilot）
upload_to_testflight

# 更新 dSYMs:
# (from itunes connect) 
download_dsyms 
upload_symbols_to_crashlytics
```

## 第二部分：HockeyApp lane

**用途：**团队日常测试

**证书类型：**Production

**provisionning profile类型：**Distribution - ad hoc

### fastlane构建过程：

1. 安装依赖
1. 运行测试
1. 更新 build number
1. 打包
1. 上传到hockeyapp 
1. 上传dSYMs至Crashlytics

### 每一步用到的fastlane命令对应如下：

``` ruby
cocoapods

scan

# 更新 build number: 
# 从jenkins的环境变量里读出build number为新的build number (ENV["BUILD_NUMBER"] ) 

gym

#（同 pilot）
upload_to_testflight 

upload_symbols_to_crashlytics
```

## 第三部分：注意事项

### 关于更新dSYMs：

> 对于testflight lane，在项目enable bitcode后，apple会对我们上传的app进行重新编译打包，因此，在本地编译的时候生成的dSYMs就没有用了，需要往crashlytics（或者其他crash report平台）上传你从iTunes Connect下载的最新的文件，因此才有了第6.1步。关于更新dSYMs，[这里](https://link.jianshu.com/?t=https%3A%2F%2Fkrausefx.com%2Fblog%2Fdownload-dsym-symbolication-files-from-itunes-connect-for-bitcode-ios-apps)有来自fastlane作者的详细的解释。

### 关于waiting for processing中断：

> 问题：由于从iTunes Connect上下载dSYMs，需要等待苹果processing这个过程完成后才能生成新的dsyms，而往往processing的过程非常漫长且时间不固定（短一点也要半小时左右）。因此，第5步 upload_to_testflight命令中，如果没有设置 skip_waiting_for_build_processing 的值为 true，则默认是false，那么这一步在上传成功后，就会一直等待processing的完成。然而processing过程太长，有可能造成命令行中断，进而造成CI失败，影响后面任务的执行。

> 解决办法：这里 **建议将  skip_waiting_for_build_processing 设置为true，同时将第6步与前面几步独立开来，放在一个单独的CI任务中执行**。

### 关于manage code signing:

> 问题：由于构建TestFlight和构建HockeyApp的包，需要的provisioning profile的类型不一样，第一个是app store类型，第二个是adhoc类型，因此，manually set provisioning profile无法同时满足两种情况。

> 解决：**建议采用（也是苹果推荐的方式）xcode automatically manage signing** 的方式，并在fastlane的gym脚本中配置不同的provisioning profile，让xcode自动去选择相应的provisioning profile文件。

> 注意：
> 如果只是在fastlane的gym命令中配置了provisioning profile，但是xcode上依然选择的是manual，最终依然会用xcode中配置的来打包，因此如果xcode配置的是adhoc的provisioning profile，但是fastlane执行的是打一个app store类型的包，就会因为类型错误而失败。

> 自动管理的方式要求在CI上用一个包含在开发team中的apple id 登录xcode，这样xcode就可以通过这个账号自动获得所有的provisioning profile了。

### 采用 TestFlight 取代 hockeyapp 作为团队的日常测试，是否可行：

> 问题：由于采用 hockeyapp 做测试分发平台，本质是用ad-hoc的方式打包和分发，因此需要注册所有测试设备的UDID。这给团队的日常管理带来了很多麻烦，尤其是每次新增测试设备，或者测试设备即将达到每年100个上限的时候，都会带来额外的工作量。还有一个问题就是，用hockeyapp测好的包，要发到app store的时候，必须重新打一个 app store类型的包，而不能把测好的包直接发布到app store，中间必然增加了出错的风险。

> 解决办法：尝试采用 TestFlight 取代 hockeyapp 作为团队的日常测试，用iTunes Connect test user apple id（内测人员ID）登录到所有测试设备的testflight中，这样每次新上传的包，不需要等待apple beta review，即可开始测试。同时，不需要UDID。

> 实测：但是实测了一下，打好的包，上传到 TestFlight 用了 8分钟左右，而TestFlight processing的过程用了 30分钟（还可能更长，不可预测）。（但是上传到 hockeyapp 只需要3分钟左右）也就是每次开发人员给测试人员出一个可测试的包，采用 TestFlight 要比 hockeyapp 多用至少35分钟。再加上编译打包等其他时间，总共得一个小时以上了。这个对于团队的日常开发测试来说基本是不可接受的。

> 结论：因此，暂时放弃了这个想法。 所以我们团队采用的是，日常story级别的测试，采用hockeyapp；上线前的PAT测试（此时需要外部人员介入帮忙一起测试），采用 TestFlight。

## 第四部分：附录

### 附录一：Fastfile  

``` ruby
fastlane_version "2.85.0"
default_platform :ios
require_relative 'actions/xctestrun'

platform :ios do
  desc "Runs all the tests"
  lane :update_dependences do
    cocoapods
  end

  desc "Runs all the tests"
  lane :test do
    unit_test
    ui_tests
  end

  desc "Build and Upload a new beta build to HockeyApp"
  lane :beta_hockeyapp do
    update_build_number_jenkins
    build_adhoc
    upload_hockeyapp
    upload_symbols_to_crashlytics(gsp_path: "./xxxxx/GoogleService-Info.plist")
  end

  desc "Build and upload a PROD app to TestFlight"
  lane :release_testflight do
    update_build_number_testflight
    build_appstore
    upload_testflight
  end

  desc "Download dSYM files from iTunes Connect and upload to Crashlytics."
  lane :upload_dsyms do
    download_dsyms(username: "XXX", app_identifier: "XXX")
    upload_symbols_to_crashlytics(gsp_path: "./xxxxx/GoogleService-Info.plist")
  end


  # -------------- private lanes below -------------- #

  # -------------- test -------------- #
  lane :ui_tests do
    xctestrun(testcase: 'xxxxxx')
    xctestrun(testcase: 'xxxxxx')
  end

  lane :unit_test do
    scan(
      workspace: "XXX",
      scheme: "XXX",
      clean: true,
      device: "iPhone 6",
      output_types: "html",
    )
  end
  # -------------- test -------------- #


  # -------------- hockeyapp -------------- #
  lane :update_build_number_jenkins do
    latestBuildNumber = ENV["BUILD_NUMBER"]
    update_info_plist(
      scheme: 'XXX',
      block: lambda { |plist|
        plist["CFBundleVersion"] = latestBuildNumber
      }
    )
  end

  lane :build_adhoc do
    gym(
      configuration: "Release",
      scheme: "XXX",
      clean: true,
      include_bitcode: true,
      include_symbols: true,
      export_method: 'ad-hoc',
      export_options: {
        provisioningProfiles: {"com.xx.xxx" => "use adhoc provisioning profile"}
      }
    )
  end

  lane :upload_hockeyapp do
    change_logs = sh "git log -5 --pretty=oneline --abbrev-commit"
    hockey(
      api_token: "XXX",
      public_identifier: "XXX",
      ipa: "XXX",
      release_type: '2',
      status: "2",
      strategy: "replace",
      notes: change_logs,
      notes_type: "1",
      bypass_cdn: true,
      timeout: 3000
    )
  end
  # -------------- hockeyapp -------------- #


  # -------------- testflight -------------- #
  lane :update_build_number_testflight do
    currentVersion = get_version_number(
      xcodeproj: "XXX"
    )
    latestBuildNumber = latest_testflight_build_number(
      username: "XXX",
      app_identifier: "XXX",
      version: currentVersion
    )
    update_info_plist(
      scheme: 'XXX',
      block: lambda { |plist|
        plist["CFBundleVersion"] = (latestBuildNumber.to_i + 1).to_s
      }
    )
  end

  lane :build_appstore do
    gym(
      configuration: "Release",
      scheme: "XXX",
      clean: true,
      include_bitcode: true,
      include_symbols: true,
      export_method: 'app-store',
      export_xcargs: "-allowProvisioningUpdates",
      export_options: {
        provisioningProfiles: {"com.xx.xxx" => "use appstore provisioning profile"}
      }
    )
  end

  lane :upload_testflight do
    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      username: "XXX",
      app_identifier: "XXX",
      ipa: "XXX")
  end
  # -------------- testflight -------------- #

end
```

## 附录二：Jenkinsfile （HockeyApp lane）

```
pipeline {
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git submodule update --init'
            }
        }

        stage('Update dependences') {
            steps {
                sh "fastlane update_dependences"
            }
        }

        stage('Run all test') {
            steps {
                sh "fastlane test"
            }
        }

        stage('Build and Upload to testflight') {
            steps {
                sh "fastlane beta_hockeyapp"
            }
        }
    }
}
```

## 附录三：Jenkinsfile （TestFlight lane）

```
pipeline {
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git submodule update --init'
            }
        }

        stage('Update dependences') {
            steps {
                sh "fastlane update_dependences"
            }
        }

        stage('Run all test') {
            steps {
                sh "fastlane test"
            }
        }

        stage('Build and Upload to testflight') {
            steps {
                sh "fastlane release_testflight"
            }
        }
    }
}
```

## 附录四：Jenkinsfile （更新dSYMs）

```
pipeline {
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git submodule update --init'
            }
        }
        stage('Update dependences') {
            steps {
                sh "fastlane upload_dsyms"
            }
        }
    }
}
```

