# KMP Publish macOS App to TestFlight

This guide walks you end-to-end through creating Apple signing assets, wiring your Compose Multiplatform (Desktop) Gradle setup, adding Fastlane lanes, and using a GitHub composite action to build & upload a macOS app to TestFlight.

## 1) Create Certificates

1. On your Mac, open Keychain Access → Keychain Access (menu) → Certificate Assistant → Request a Certificate From a Certificate Authority…

    Enter your email and a Common Name (e.g., your org or your name).
    Select Saved to disk, then Continue to create a CSR file.
    You’ll attach this CSR on Apple Developer when creating certificates.

2. Go to Apple Developer → Certificates:
https://developer.apple.com/account/resources/certificates/list

Create two certificates:

- Mac App Distribution
  Used to code sign your app and for distribution profiles.

- Mac Installer Distribution
  Used to sign the Installer Package (.pkg).

For both, upload the CSR you created above.

3. Download and install the generated certificates
(double click or drag into Keychain Access).

4. Verify they installed:
```
/usr/bin/security find-certificate -c "3rd Party Mac Developer Application"
/usr/bin/security find-certificate -c "3rd Party Mac Developer Installer"
```

## 2) Create App IDs (2 of them)

An App ID represents one or more apps in Apple’s ecosystem.

1. Visit the App IDs page in the developer portal.

2. Create new App ID → App type → fill Bundle ID (reverse-DNS, e.g. com.yoursite.yourapp).

3. Create another App ID for the runtime using the same pattern but prefixed:

- App ID (app): com.yoursite.yourapp (this is your bundle ID)

- App ID (runtime): com.oracle.java.com.yoursite.yourapp

The runtime App ID is required for TestFlight/App Store distribution of JVM-based apps.

## 3) Create Provisioning Profiles (2 of them)

You need two Mac App Store (Distribution) provisioning profiles:

1. On the Apple Developer portal → Profiles → +

2. Distribution → Mac App Store → Mac

3. Select your App ID (app) → select your Mac App Distribution certificate → name it
e.g., AppStore App com.yoursite.yourapp → Generate → Download.

4. Repeat for the runtime App ID (com.oracle.java.com.yoursite.yourapp), e.g.
AppStore Runtime com.oracle.java.com.yoursite.yourapp.

You’ll end up with two .provisionprofile files.

## 4) Configure Gradle (Compose Desktop)
1. Bundle ID
```
macOS {
    bundleID = "com.example-company.example-app" // must match your App ID
}
```


Rules

- Alphanumeric, - and . allowed.

- Use reverse-DNS (e.g., com.yoursite.yourapp).

- Must match one of your App IDs.

2. Signing
```
macOS {
    signing {
        sign.set(true) // or -Pcompose.desktop.mac.sign=true
        // Use the EXACT identity shown in Keychain:
        // "3rd Party Mac Developer Application: <Team Name> (<TEAM_ID>)"
        identity.set("The Mifos Initiative")
        // keychain.set("/path/to/keychain") // optional when multiple similar certs exist
    }
}
```

Set identity to the Team Name.

3. Provisioning Profiles

Requires JDK 18+ (Compose Desktop runtime provisioning support).

Place the two profiles in your desktop module directory (and add to .gitignore to avoid committing):

- embedded.provisionprofile (app)

- runtime.provisionprofile (runtime)

```
macOS {
    provisioningProfile.set(project.file("embedded.provisionprofile"))
    runtimeProvisioningProfile.set(project.file("runtime.provisionprofile"))
}
```

4. Entitlements

Create entitlements.plist:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key><true/>
    <key>com.apple.security.cs.allow-jit</key><true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
    <key>com.apple.security.cs.disable-library-validation</key><true/>
    <key>com.apple.security.cs.allow-dyld-environment-variables</key><true/>
    <key>com.apple.security.cs.debugger</key><true/>
    <key>com.apple.security.device.audio-input</key><true/>
    <key>com.apple.application-identifier</key><string>TEAMID.APPID</string>
    <key>com.apple.developer.team-identifier</key><string>TEAMID</string>
    <!-- Add more entitlements as needed -->
</dict>
</plist>

```

Create runtime-entitlements.plist:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key><true/>
    <key>com.apple.security.cs.allow-jit</key><true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
    <key>com.apple.security.cs.disable-library-validation</key><true/>
    <key>com.apple.security.cs.allow-dyld-environment-variables</key><true/>
    <key>com.apple.security.cs.debugger</key><true/>
    <key>com.apple.security.device.audio-input</key><true/>
</dict>
</plist>
```


Then wire them in Gradle:
```
macOS {
    entitlementsFile.set(project.file("entitlements.plist"))
    runtimeEntitlementsFile.set(project.file("runtime-entitlements.plist"))
}
```

Push both plist files to your repo.

5. Full Gradle Example
```
import org.jetbrains.compose.desktop.application.dsl.TargetFormat

compose.desktop {
    mainClass = "example.MainKt"

    val buildNumber: String = (project.findProperty("buildNumber") as String?) ?: "1"
    val isAppStoreRelease: Boolean =
        (project.findProperty("macOsAppStoreRelease") as String?)?.toBoolean() ?: false

    nativeDistributions {
        targetFormats(TargetFormat.Pkg /*, … */)

        // Fill these with your app’s details
        packageName = "Your App Name"
        packageVersion = "1.0.0"
        description = "..."
        copyright = "..."
        vendor = "..."
        licenseFile.set(project.file("../LICENSE"))
        includeAllModules = true
        outputBaseDir.set(project.layout.buildDirectory.dir("release"))

        macOS {
            bundleID = "com.yoursite.yourapp"
            dockName = "Your App Name"
            iconFile.set(project.file("icons/ic_launcher.icns"))
            minimumSystemVersion = "12.0"
            appStore = isAppStoreRelease

            infoPlist {
                packageBuildVersion = buildNumber
                extraKeysRawXml = """
                    <key>ITSAppUsesNonExemptEncryption</key>
                    <false/>
                """.trimIndent()
            }

            if (isAppStoreRelease) {
                signing {
                    sign.set(true)
                    identity.set("The Mifos Initiative")
                }
                provisioningProfile.set(project.file("embedded.provisionprofile"))
                runtimeProvisioningProfile.set(project.file("runtime.provisionprofile"))
                entitlementsFile.set(project.file("entitlements.plist"))
                runtimeEntitlementsFile.set(project.file("runtime-entitlements.plist"))
            } else {
                notarization {
                    val providers = project.providers
                    appleID.set(providers.environmentVariable("NOTARIZATION_APPLE_ID"))
                    password.set(providers.environmentVariable("NOTARIZATION_PASSWORD"))
                    teamID.set(providers.environmentVariable("NOTARIZATION_TEAM_ID"))
                }
            }
        }

        windows {
            menuGroup = "YourApp"
            shortcut = true
            dirChooser = true
            perUserInstall = true
            iconFile.set(project.file("icons/ic_launcher.ico"))
        }
        linux {
            modules("jdk.security.auth")
            iconFile.set(project.file("icons/ic_launcher.png"))
        }
    }
}
```

6. (Recommended) Un-quarantine Task
   Sometimes files pulled during build (JBR/JRE, libs) carry the com.apple.quarantine xattr and App Store will reject. This task removes that attribute before signing/packaging:
```
/**
 * Removes the `com.apple.quarantine` extended attribute from the built `.app`.
 *
 * Why:
 * Gatekeeper may mark files from the Internet with `com.apple.quarantine`.
 * If any such file ends up inside the `.app`, App Store validation can fail.
 */
val unquarantineApp = tasks.register<Exec>("unquarantineMacApp") {
    group = "macOS"
    description = "Remove com.apple.quarantine from the built .app before signing"
    onlyIf { org.gradle.internal.os.OperatingSystem.current().isMacOsX }

    dependsOn("createReleaseDistributable")

    val appName = "YourApp.app" // set to your final .app name
    val appPath = layout.buildDirectory
        .dir("release/main-release/app/$appName")
        .map { it.asFile.absolutePath }

    commandLine("xattr", "-dr", "com.apple.quarantine", appPath.get())
}

tasks.matching { it.name == "packageReleasePkg" }.configureEach {
    dependsOn(unquarantineApp)
}
```

## 5) App Store Connect API Key

1. App Store Connect → Users and Access → Keys → +

2. You’ll receive: Key ID, Issuer ID, Private Key (.p8)

For local use, you can place it at: secrets/Api_key.p8 (and add to .gitignore).
For CI, store the Base64 contents in a secret.

## 6) Fastlane Setup
   Install (once)

- Install Ruby: https://www.ruby-lang.org/en/documentation/installation/

- Install Fastlane: https://docs.fastlane.tools/getting-started/ios/setup/

Verify:
```
ruby -v
fastlane -v
```

Initialize in your project root:
```
fastlane init
```

This creates:
```
rootProject/
└── fastlane/
    ├── Appfile
    └── Fastfile
```

Fastfile
```yaml
platform :mac do
     desc "Build & upload macOS (.pkg) to TestFlight"
     lane :desktop_testflight do |options|

        # Resolve inputs (CLI options → default fallbacks)
        app_identifier = options[:app_identifier] || 'com.example.yourapp'
        cmp_desktop_dir = options[:cmp_desktop_dir] || 'cmp-desktop'
    
        appstore_key_id    = options[:appstore_key_id]    || '5G4JABCN52'
        appstore_issuer_id = options[:appstore_issuer_id] || '9af9e161-9603-2k3e-h137-ae521f816088'
        key_file_path      = options[:key_file_path]      || 'secrets/Api_key.p8'

        # CI setup (only when on CI)
        if ENV['CI']
          setup_ci
        else
          UI.message("🖥️ Running locally, skipping CI-specific setup.")
        end
    
        # App Store Connect API key (JWT)
        api_key = app_store_connect_api_key(
          key_id:      appstore_key_id,
          issuer_id:   appstore_issuer_id,
          key_filepath: key_file_path,
          duration: 1200 # seconds, ~20 minutes
        )

        # Determine next build number from TestFlight (macOS)
        latest = latest_testflight_build_number(
          app_identifier: app_identifier,
          api_key: api_key,
          platform: 'osx',
          version: '1.0.0'
        )
        new_build_number = (latest.to_i + 1).to_s
        UI.message("Next build number: #{new_build_number}")
    
        # Build the signed .pkg via Gradle / Compose
        gradle(
          tasks: ['packageReleasePkg'],
          properties: {
            'buildNumber' => new_build_number,
            'macOsAppStoreRelease' => true
          }
        )
    
        # Locate the most recent generated .pkg artifact
        project_dir = File.expand_path('..', Dir.pwd) # fastlane/ -> project root
        pkg_glob = File.join(project_dir, cmp_desktop_dir, 'build', 'release', '**', 'pkg', '*.pkg')
        candidates = Dir[pkg_glob]
        UI.user_error!("PKG not found! Looked for: #{pkg_glob}") if candidates.empty?
    
        pkg_path = candidates.max_by { |p| File.mtime(p) }
        UI.message("Found PKG at: #{pkg_path}")
    
        # Upload the .pkg to TestFlight (pilot)
        pilot(
          api_key: api_key,
          pkg: pkg_path,
          app_platform: 'osx',
          skip_waiting_for_build_processing: true
        )
    end
end
```

Run locally:
```
bundle exec fastlane mac desktop_testflight
```

## 7) GitHub Secrets (what to add)
   Export & Base64-encode your certificates

In Keychain Access → My Certificates, export:

- 3rd Party Mac Developer Application: <Team Name> (<TEAM_ID>) → MAC_APP_DISTRIBUTION_CERTIFICATE.p12

- 3rd Party Mac Developer Installer: <Team Name> (<TEAM_ID>) → MAC_INSTALLER_DISTRIBUTION_CERTIFICATE.p12

Set a strong export password (you’ll use it as CERTIFICATES_PASSWORD).

Encode to Base64 (so you can store in Secrets):

```
# macOS: copy Base64 to clipboard
base64 -i MAC_APP_DISTRIBUTION_CERTIFICATE.p12 | pbcopy
# paste into secret: MAC_APP_DISTRIBUTION_CERTIFICATE_B64

base64 -i MAC_INSTALLER_DISTRIBUTION_CERTIFICATE.p12 | pbcopy
# paste into secret: MAC_INSTALLER_DISTRIBUTION_CERTIFICATE_B64
```

Base64-encode provisioning profiles
```
base64 -i embedded.provisionprofile | pbcopy     # -> MAC_EMBEDDED_PROVISION_B64
base64 -i runtime.provisionprofile  | pbcopy     # -> MAC_RUNTIME_PROVISION_B64
```

Other secrets to create

Secret | Description
-- | --
KEYCHAIN_PASSWORD | Any strong password for the temporary CI keychain
CERTIFICATES_PASSWORD |	The export password you set for both .p12 files
APPSTORE_KEY_ID	App | Store Connect API Key ID
APPSTORE_ISSUER_ID | App Store Connect Issuer ID
APPSTORE_AUTH_KEY |	Base64 of your .p8 private key

## 8) GitHub Actions Workflow (using the composite action)

Example workflow that builds and uploads to TestFlight.

```yaml
name: macOS Build & Distribute (TestFlight / App Store)

on:
  workflow_dispatch:

jobs:
  ship_mac:
    runs-on: macos-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Build & Upload to TestFlight
        uses: mifos-x-actionhub-publish-macos-on-appstore-testflight@main
        with:
          app_identifier: com.example.yourapp
          cmp_desktop_dir: cmp-desktop                        # REQUIRED
          keychain_name: signing-${{ github.run_id }}.keychain-db   # optional (default provided by action)
          java_version: '21'                                   # optional (default 21, minimum 18)
          keychain_password: ${{ secrets.KEYCHAIN_PASSWORD }}
          certificates_password: ${{ secrets.CERTIFICATES_PASSWORD }}
          mac_app_distribution_certificate_b64: ${{ secrets.MAC_APP_DISTRIBUTION_CERTIFICATE_B64 }}
          mac_installer_distribution_certificate_b64: ${{ secrets.MAC_INSTALLER_DISTRIBUTION_CERTIFICATE_B64 }}
          mac_embedded_provision_b64: ${{ secrets.MAC_EMBEDDED_PROVISION_B64 }}
          mac_runtime_provision_b64: ${{ secrets.MAC_RUNTIME_PROVISION_B64 }}
          appstore_key_id: ${{ secrets.APPSTORE_KEY_ID }}
          appstore_issuer_id: ${{ secrets.APPSTORE_ISSUER_ID }}
          appstore_auth_key_b64: ${{ secrets.APPSTORE_AUTH_KEY }}
```