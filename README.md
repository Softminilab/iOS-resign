## 主要使用的命令

1. 签名框架和插件 (如果有)： 如果 .app 文件中包含框架 (Frameworks 文件夹)、插件 (PlugIns 文件夹) 或应用扩展 (AppExtensions 文件夹)，你需要先对它们进行签名。

   ```Shell
   codesign -f -s "你的证书名称" --entitlements entitlements.plist path/to/Payload/YourApp.app/Frameworks/SomeFramework.framework
   
   # 对每个 framework, plugin, appex 都执行类似操作
   ```

   

2. 提取entitlements.plist

   ```Shell
   security cms -D -i embedded.mobileprovision > temp.plist
   /usr/libexec/PlistBuddy -x -c 'Print :Entitlements' temp.plist > entitlements.plist
   rm temp.plist
   ```

3. 签名主程序

   ```Shell
   codesign -f -s "你的证书名称" --entitlements entitlements.plist path/to/Payload/YourApp.app
   ```

   

4. 打包zip

   ```Shell
   zip -qr NewAppName.ipa Payload/
   ```



## 具体步骤如下

是的，可以下载一个IPA文件然后使用你自己的本地（Apple颁发的）证书进行重新签名。这通常用于以下几种情况：

1.  **企业内部分发**：使用企业开发者证书重新签名应用，以便在组织内部设备上安装。
2.  **Ad Hoc分发**：使用Ad Hoc证书重新签名，以便在注册的测试设备上安装。
3.  **个人使用/测试**：使用免费或付费的个人开发者证书重新签名，以便在自己的设备上安装（免费证书有7天有效期限制）。

**重要前提：**
*   你需要在你的Mac上拥有有效的Apple开发者证书（例如 "iPhone Developer: Your Name (TEAMID)" 或 "Apple Development: Your Name (TEAMID)"）及其对应的私钥（存储在钥匙串中）。
*   你需要一个与该证书关联的有效**置备描述文件 (Provisioning Profile)**。这个描述文件需要：
    *   包含你要签名的App的Bundle ID（或者是一个通配符Bundle ID，如 `com.yourteam.*`，但通配符不支持某些高级功能如推送通知、App Groups等）。
    *   包含你要安装应用的设备的UDID（对于Ad Hoc和免费开发者证书）。
    *   链接到你的开发者证书。
*   你的Mac上需要安装Xcode（为了Xcode Command Line Tools，特别是 `codesign` 和 `security` 工具）。

**具体步骤（使用命令行）：**

这是一个相对复杂的过程，需要一定的技术背景。

1.  **准备工作：**
    *   将你的IPA文件（例如 `OriginalApp.ipa`）放在一个工作目录中。
    *   下载与你的证书和App ID匹配的置备描述文件（例如 `YourProfile.mobileprovision`）到同一目录。

2.  **解压IPA文件：**
    IPA文件本质上是一个ZIP压缩包。
    ```bash
    unzip OriginalApp.ipa -d PayloadDir
    # 这会在PayloadDir目录下创建一个Payload文件夹，里面是 .app 文件，例如 Payload/OriginalApp.app
    ```
    或者，你可以直接将 `.ipa` 后缀改为 `.zip` 然后用Finder解压。

3.  **移除旧的签名文件：**
    进入到 `.app` 包内，删除旧的签名信息。
    ```bash
    rm -rf PayloadDir/Payload/OriginalApp.app/_CodeSignature
    rm -rf PayloadDir/Payload/OriginalApp.app/Frameworks/*/_CodeSignature # 如果有Frameworks
    rm -rf PayloadDir/Payload/OriginalApp.app/PlugIns/*/_CodeSignature    # 如果有App Extensions
    rm -rf PayloadDir/Payload/OriginalApp.app/Watch/*/_CodeSignature      # 如果有Watch App
    ```

4.  **替换置备描述文件：**
    将你准备好的置备描述文件复制到 `.app` 包内，并命名为 `embedded.mobileprovision`。
    ```bash
    cp YourProfile.mobileprovision PayloadDir/Payload/OriginalApp.app/embedded.mobileprovision
    ```

5.  **（可选但通常必要）修改Bundle Identifier：**
    你的置备描述文件是针对特定Bundle ID（或通配符）的。如果原始IPA的Bundle ID与你的描述文件不匹配，你需要修改 `.app` 包内的 `Info.plist` 文件。
    *   找到 `PayloadDir/Payload/OriginalApp.app/Info.plist`。
    *   使用文本编辑器或`PlistBuddy`修改 `CFBundleIdentifier` 键的值。
    ```bash
    # 假设你的新Bundle ID是 com.yourcompany.newappid
    /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier com.yourcompany.newappid" PayloadDir/Payload/OriginalApp.app/Info.plist
    ```
    如果你有App Extensions或Watch App，它们各自的 `Info.plist` 中的Bundle ID也可能需要修改，通常是主App Bundle ID的扩展（例如 `com.yourcompany.newappid.widget`）。

6.  **获取签名身份 (Signing Identity)：**
    在终端中列出你可用的签名身份：
    ```bash
    security find-identity -v -p codesigning
    ```
    你会看到类似 `"Apple Development: Your Name (XXXXXXXXXX)"` 或 `"iPhone Distribution: Your Company (YYYYYYYYYY)"` 的列表。复制你想要使用的那个身份的全名（包括引号内的部分）。我们称之为 `CERTIFICATE_NAME`。

7.  **准备Entitlements（权限）：**
    通常，你需要从新的置备描述文件中提取entitlements，或者创建一个。
    ```bash
    # 从置备描述文件提取entitlements到一个临时plist
    security cms -D -i PayloadDir/Payload/OriginalApp.app/embedded.mobileprovision > temp_profile.plist
    # 从临时plist中提取entitlements部分到 entitlements.plist
    /usr/libexec/PlistBuddy -x -c 'Print :Entitlements' temp_profile.plist > entitlements.plist
    # 清理临时文件
    rm temp_profile.plist
    ```
    **注意**：如果原始应用使用了你的描述文件不支持的特殊权限（例如，原始应用用了HealthKit，但你的描述文件没配置HealthKit权限），签名可能会成功，但应用运行时会崩溃或功能不正常。你需要确保entitlements.plist中的权限与你的描述文件和证书能力相符。

8.  **重新签名：**
    你需要按特定顺序签名：任何动态库 (Frameworks)、插件 (App Extensions)、Watch App，最后才是主应用。
    ```bash
    # 签名Frameworks (如果有)
    # find PayloadDir/Payload/OriginalApp.app/Frameworks -name "*.framework" -exec codesign -f -s "${CERTIFICATE_NAME}" --entitlements entitlements.plist {} \;
    # 更可靠的方式是逐个签名，如果它们有自己的可执行文件
    if [ -d "PayloadDir/Payload/OriginalApp.app/Frameworks" ]; then
        for FRAMEWORK in $(find PayloadDir/Payload/OriginalApp.app/Frameworks -name "*.framework"); do
            echo "Signing framework: ${FRAMEWORK}"
            codesign -f -s "${CERTIFICATE_NAME}" --entitlements entitlements.plist "${FRAMEWORK}"
        done
    fi
    
    # 签名App Extensions (PlugIns, 如果有)
    if [ -d "PayloadDir/Payload/OriginalApp.app/PlugIns" ]; then
        for PLUGIN in $(find PayloadDir/Payload/OriginalApp.app/PlugIns -name "*.appex"); do
            echo "Signing plugin: ${PLUGIN}"
            # 插件可能需要自己的entitlements，或者使用主应用的，取决于插件类型
            codesign -f -s "${CERTIFICATE_NAME}" --entitlements entitlements.plist "${PLUGIN}"
        done
    fi
    
    # 签名Watch App (如果有)
    if [ -d "PayloadDir/Payload/OriginalApp.app/Watch" ]; then
        for WATCHAPP in $(find PayloadDir/Payload/OriginalApp.app/Watch -name "*.app"); do
            echo "Signing watch app: ${WATCHAPP}"
            # Watch App也可能有自己的Frameworks或Extensions，需要先签名它们
            codesign -f -s "${CERTIFICATE_NAME}" --entitlements entitlements.plist "${WATCHAPP}"
        done
    fi
    
    # 最后签名主应用
    echo "Signing main application"
    codesign -f -s "${CERTIFICATE_NAME}" --entitlements entitlements.plist PayloadDir/Payload/OriginalApp.app
    ```
    **注意：** `${CERTIFICATE_NAME}` 要替换为你从步骤6中获取的实际证书名称。`-f` 参数表示强制签名。

9.  **重新打包成IPA：**
    ```bash
    cd PayloadDir
    zip -qr ../NewSignedApp.ipa Payload
    cd ..
    ```
    这会在你的工作目录下创建一个 `NewSignedApp.ipa` 文件。

10. **安装：**
    现在你可以通过Xcode (Window > Devices and Simulators，然后将IPA拖到设备上) 或Apple Configurator 2等工具将 `NewSignedApp.ipa` 安装到你的设备上（设备UDID必须在置备描述文件中）。

**简化工具：**
这个手动过程非常繁琐且容易出错。有一些第三方工具可以简化这个过程：
*   **iOS App Signer**：一个macOS图形界面工具，可以自动完成大部分步骤。
*   **AltStore/Sideloadly**：主要用于侧载应用，它们内部也执行了重签名过程，通常使用免费开发者证书。
*   **fastlane (resign action)**：如果你熟悉fastlane，它有一个 `resign` action 可以自动化这个过程。

**重要注意事项：**
*   **法律和道德**：确保你有权重新分发或修改你正在签名的应用。从App Store下载的应用通常有版权保护，未经授权的修改和分发可能违法。
*   **Entitlements 兼容性**：这是最棘手的部分。如果原始应用的entitlements与你的证书和置备描述文件不兼容（例如，应用需要iCloud，但你的描述文件不允许），应用可能无法正确签名，或者签名后运行时崩溃。
*   **App Thinning**：从App Store下载的IPA可能是针对特定设备架构（如arm64）进行了“瘦身”的。如果你尝试在不兼容的设备上安装，可能会失败。最好使用开发者提供的“通用”IPA包。
*   **免费开发者证书**：使用免费开发者证书签名的应用有效期只有7天，并且每年只能在有限数量的设备上安装。

总的来说，虽然技术上可行，但过程复杂，特别是entitlements的处理。对于普通用户，使用如iOS App Signer这样的工具会更容易。





## Codesign 常用相关命令

`codesign` 是 macOS 上一个非常强大的命令行工具，用于创建、检查和管理代码签名。以下是一些常用的 `codesign` 命令及其说明：

**1. 签名代码 (Signing Code)**

*   **基本签名:**
    ```bash
    codesign -s "Signing Identity Name" /path/to/YourApp.app
    ```
    *   `-s "Signing Identity Name"`: 指定用于签名的证书。 `"Signing Identity Name"` 是你在钥匙串中看到的证书全名（例如 "Apple Development: Your Name (TEAMID)" 或 "iPhone Distribution: Your Company (TEAMID)"）。你可以通过 `security find-identity -v -p codesigning` 命令找到可用的身份。
    *   `/path/to/YourApp.app`: 要签名的应用程序包、框架、插件或其他可执行文件的路径。

*   **强制签名 (覆盖现有签名):**
    ```bash
    codesign -f -s "Signing Identity Name" /path/to/YourApp.app
    ```
    *   `-f` 或 `--force`: 如果目标已经签名，则强制替换现有签名。

*   **深度签名 (签名所有嵌套内容):**
    ```bash
    codesign -f -s "Signing Identity Name" --deep /path/to/YourApp.app
    ```
    *   `--deep`: 签名应用程序包时，此选项会递归地签名所有嵌套的可执行代码，如框架、插件、XPC服务等。**注意：** 官方推荐分别签名每个组件，而不是过度依赖 `--deep`，因为 `--deep` 可能无法正确处理某些复杂情况或不同的entitlements。但在简单情况下，它很方便。

*   **使用特定Entitlements文件签名:**
    ```bash
    codesign -f -s "Signing Identity Name" --entitlements /path/to/entitlements.plist /path/to/YourApp.app
    ```
    *   `--entitlements /path/to/entitlements.plist`: 指定一个 `.plist` 文件，其中包含应用程序应有的权限（entitlements）。这对于沙盒应用、iCloud、推送通知等功能至关重要。

**2. 验证签名 (Verifying Signatures)**

*   **基本验证:**
    ```bash
    codesign --verify /path/to/YourApp.app
    ```
    *   `--verify`: 检查签名是否有效。如果签名有效且满足所有系统策略，则此命令成功（不输出任何内容）。如果签名无效或有问题，它将打印错误消息并以非零状态退出。

*   **详细验证:**
    ```bash
    codesign -v --verify /path/to/YourApp.app
    # 或者更详细
    codesign -vvvv --verify /path/to/YourApp.app
    ```
    *   `-v` 或 `--verbose`: 提供更详细的输出。可以多次使用（例如 `-vv`, `-vvv`, `-vvvv`）以增加详细程度。

*   **检查是否满足特定要求 (Advanced):**
    ```bash
    codesign --verify --requirements "designated => anchor apple generic and ..." /path/to/YourApp.app
    ```
    *   `--requirements "..."`: 根据指定的代码签名需求集来验证签名。这通常用于更高级的验证场景。

**3. 显示签名信息 (Displaying Signature Information)**

*   **显示基本签名信息:**
    ```bash
    codesign -d /path/to/YourApp.app
    # 或者更常用且更详细
    codesign -dv /path/to/YourApp.app
    ```
    *   `-d` 或 `--display`: 显示关于签名的信息。
    *   `-v` (与 `-d` 一起使用时): 增加显示信息的详细程度。

*   **显示非常详细的签名信息:**
    ```bash
    codesign -dvvvv /path/to/YourApp.app
    # 或者
    codesign -d --verbose=4 /path/to/YourApp.app
    ```
    *   这会显示大量信息，包括签名者、时间戳、Team ID、CDHash、Entitlements 等。

*   **仅显示Entitlements:**
    ```bash
    codesign -d --entitlements :- /path/to/YourApp.app
    ```
    *   `--entitlements :-`: 将嵌入的entitlements以XML plist格式打印到标准输出（`:-` 表示标准输出）。
    *   你可以将其重定向到文件：
        ```bash
        codesign -d --entitlements :- /path/to/YourApp.app > extracted_entitlements.plist
        ```

*   **显示CDHash (Code Directory Hash):**
    ```bash
    codesign -dvvvv /path/to/YourApp.app | grep "CDHash="
    # 或者更精确的方式，如果知道格式
    ```
    CDHash 对于创建 LaunchDaemon/LaunchAgent 配置文件中的 `ProgramArguments` 来指定可信程序非常有用。

**4. 移除签名 (Removing Signatures)**

*   **移除签名 (不常用，主要用于调试或重新签名准备):**
    ```bash
    codesign --remove-signature /path/to/YourApp.app
    ```
    *   `--remove-signature`: 从指定的代码中移除最外层的签名。

**5. 查找签名身份 (配合 `security` 工具)**

虽然这不是 `codesign` 的直接命令，但与签名过程密切相关：
```bash
security find-identity -v -p codesigning
```
*   此命令列出你钥匙串中所有可用于代码签名的有效身份（证书及其私钥）。你需要从这个列表中选择正确的身份名称（例如 `"Apple Development: John Doe (ABC123XYZ)"`）用于 `codesign -s` 命令。

**重要提示:**

*   **Xcode Command Line Tools**: `codesign` 是 Xcode Command Line Tools 的一部分。你需要安装它们才能使用此命令。
*   **钥匙串访问**: 你的签名证书及其对应的私钥必须存在于你的登录钥匙串中，并且可供 `codesign` 访问。
*   **顺序**: 当签名包含多个可执行组件的应用程序包时（例如，应用包含框架或插件），通常需要先签名内部组件，最后签名主应用包。
*   **权限**: 确保你对要签名或修改的文件/目录有适当的写入权限。

这些是最常用的 `codesign` 命令。根据具体需求，还有更多高级选项和用法，可以通过 `man codesign` 查看完整的帮助文档。