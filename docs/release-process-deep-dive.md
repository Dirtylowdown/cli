# Release Process Deep Dive

## Motivation

The current release workflow and associated scripts were created before the current set of maintainers, and all maintainers from that time have left. On a number of occasions (releasing a MacOS installer, moving to Azure HCS signing, updating expired GPG key) the current maintainers have spent time investigating the release workflow. This document is intended to serve as a guide for future maintainers who need to understand the release process.

## High Level Overview

From a high level, the release workflow:
 * Is triggered by a `workflow_dispatch` event (typically a result of running `./script/release`)
 * Builds, packages and signs artifacts in parallel for Linux, MacOS and Windows
 * GPG signs our Debian and Red Hat repository artifacts
 * Builds and updates the [marketing site](https://cli.github.com), [manual](https://cli.github.com/manual) and repo packages
 * Creates GitHub Attestations for the artifacts
 * Creates a GitHub Release and attaches the artifacts
 * Bumps the `gh` [homebrew-core formula](https://github.com/Homebrew/homebrew-core/blob/2df031cbd8f7bc9b9a380e941ccefcf3c8f3d02b/Formula/g/gh.rb)

## Jobs Deep Dive

This section will deep dive into each job in the [`deployment.yml` workflow](https://github.com/cli/cli/blob/537a22228cd6b42b740d7f1c09f47c45bb1dab30/.github/workflows/deployment.yml).

// TODO Screenshot of the deployment workflow

- [validate-tag-name](#validate-tag-name)
- [linux](#linux)
- [macos](#macos)
- [windows](#windows)
- [release](#release)

### <a id="validate-tag-name">[validate-tag-name](https://github.com/cli/cli/blob/537a22228cd6b42b740d7f1c09f47c45bb1dab30/.github/workflows/deployment.yml#L31-L39)</a>

<details>

```yml
  validate-tag-name:
    runs-on: ubuntu-latest
    steps:
      - name: Validate tag name format
        run: |
          if [[ ! "${{ inputs.tag_name }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Invalid tag name format. Must be in the form v1.2.3"
            exit 1
          fi
```
</details>

The purpose of this job is to prevent releases that are tagged incorrectly, by ensuring they conform to the major.minor.patch form of semantic versioning, preceeded by a `v`. [Reference](https://github.com/cli/cli/pull/10121).

### <a id="linux">linux</a>

<details>

```yml
  linux:
    needs: validate-tag-name
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    if: contains(inputs.platforms, 'linux')
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: 'go.mod'
      - name: Install GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          version: "~1.17.1"
          install-only: true
      - name: Build release binaries
        env:
          TAG_NAME: ${{ inputs.tag_name }}
        run: script/release --local "$TAG_NAME" --platform linux
      - name: Generate web manual pages
        run: |
          go run ./cmd/gen-docs --website --doc-path dist/manual
          tar -czvf dist/manual.tar.gz -C dist -- manual
      - uses: actions/upload-artifact@v4
        with:
          name: linux
          if-no-files-found: error
          retention-days: 7
          path: |
            dist/*.tar.gz
            dist/*.rpm
            dist/*.deb
```
</details>

The purpose of this job is to:
 * Build the executables for linux, archive (`tar.gz`) and package (`.deb`, `.rpm`) them using [`GoReleaser`](https://goreleaser.com/)
 * Build the [CLI manual](https://cli.github.com/manual/) on behalf of all operating systems

Deeper dives can be found for:
 * [./script/release](#script-release)
 * [GoReleaser](#goreleaser)

### <a id="macos">macos</a>

<details>

```yaml
  macos:
    needs: validate-tag-name
    runs-on: macos-latest
    environment: ${{ inputs.environment }}
    if: contains(inputs.platforms, 'macos')
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: 'go.mod'
      - name: Configure macOS signing
        if: inputs.environment == 'production'
        env:
          APPLE_DEVELOPER_ID: ${{ vars.APPLE_DEVELOPER_ID }}
          APPLE_APPLICATION_CERT: ${{ secrets.APPLE_APPLICATION_CERT }}
          APPLE_APPLICATION_CERT_PASSWORD: ${{ secrets.APPLE_APPLICATION_CERT_PASSWORD }}
        run: |
          keychain="$RUNNER_TEMP/buildagent.keychain"
          keychain_password="password1"

          security create-keychain -p "$keychain_password" "$keychain"
          security default-keychain -s "$keychain"
          security unlock-keychain -p "$keychain_password" "$keychain"

          base64 -D <<<"$APPLE_APPLICATION_CERT" > "$RUNNER_TEMP/cert.p12"
          security import "$RUNNER_TEMP/cert.p12" -k "$keychain" -P "$APPLE_APPLICATION_CERT_PASSWORD" -T /usr/bin/codesign
          security set-key-partition-list -S "apple-tool:,apple:,codesign:" -s -k "$keychain_password" "$keychain"
          rm "$RUNNER_TEMP/cert.p12"
      - name: Install GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          version: "~1.17.1"
          install-only: true
      - name: Build release binaries
        env:
          TAG_NAME: ${{ inputs.tag_name }}
          APPLE_DEVELOPER_ID: ${{ vars.APPLE_DEVELOPER_ID }}
        run: script/release --local "$TAG_NAME" --platform macos
      - name: Notarize macOS archives
        if: inputs.environment == 'production'
        env:
          APPLE_ID: ${{ vars.APPLE_ID }}
          APPLE_ID_PASSWORD: ${{ secrets.APPLE_ID_PASSWORD }}
          APPLE_DEVELOPER_ID: ${{ vars.APPLE_DEVELOPER_ID }}
        run: |
          shopt -s failglob
          script/sign dist/gh_*_macOS_*.zip
      - name: Build universal macOS pkg installer
        if: inputs.environment != 'production'
        env:
          TAG_NAME: ${{ inputs.tag_name }}
        run: script/pkgmacos "$TAG_NAME"
      - name: Build & notarize universal macOS pkg installer
        if: inputs.environment == 'production'
        env:
          TAG_NAME: ${{ inputs.tag_name }}
          APPLE_DEVELOPER_INSTALLER_ID: ${{ vars.APPLE_DEVELOPER_INSTALLER_ID }}
        run: |
          shopt -s failglob
          script/pkgmacos "$TAG_NAME"
      - uses: actions/upload-artifact@v4
        with:
          name: macos
          if-no-files-found: error
          retention-days: 7
          path: |
            dist/*.tar.gz
            dist/*.zip
            dist/*.pkg
```
</details>

The purpose of this job is to:
 * Build the executables for MacOS and archive them (`.zip`) using [`GoReleaser`](https://goreleaser.com/)
 * Notarize the archives
 * Build and Notarize (TODO: doesn't ever happen) the MacOS Installer

Deeper dives can be found for:
 * [./script/release](#script-release)
 * [GoReleaser](#goreleaser)
 * [./script/sign](#script-sign)
 * [./script/pkgmacos](#script-pkgmacos)

### <a id="windows">windows</a>

<details>

```yml
windows:
    needs: validate-tag-name
    runs-on: windows-latest
    environment: ${{ inputs.environment }}
    if: contains(inputs.platforms, 'windows')
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: 'go.mod'
      - name: Install GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          version: "~1.17.1"
          install-only: true
      - name: Install Azure Code Signing Client
        shell: pwsh
        env:
          ACS_DIR: ${{ runner.temp }}\acs
          ACS_ZIP: ${{ runner.temp }}\acs.zip
          CORRELATION_ID: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          METADATA_PATH: ${{ runner.temp }}\acs\metadata.json
        run: |
          # Download Azure Code Signing client containing the DLL needed for signtool in script/sign
          Invoke-WebRequest -Uri https://www.nuget.org/api/v2/package/Azure.CodeSigning.Client/1.0.43 -OutFile $Env:ACS_ZIP -Verbose
          Expand-Archive $Env:ACS_ZIP -Destination $Env:ACS_DIR -Force -Verbose

          # Generate metadata file for signtool, used in signing box .exe and .msi
          @{
            CertificateProfileName = "GitHubInc"
            CodeSigningAccountName = "GitHubInc"
            CorrelationId = $Env:CORRELATION_ID
            Endpoint =  "https://wus.codesigning.azure.net/"
          } | ConvertTo-Json | Out-File -FilePath $Env:METADATA_PATH

      # Azure Code Signing leverages the environment variables for secrets that complement the metadata.json
      # file generated above (AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID)
      # For more information, see https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential?view=azure-dotnet
      - name: Build release binaries
        shell: bash
        env:
          AZURE_CLIENT_ID: ${{ secrets.SPN_GITHUB_CLI_SIGNING_CLIENT_ID }}
          AZURE_CLIENT_SECRET: ${{ secrets.SPN_GITHUB_CLI_SIGNING }}
          AZURE_TENANT_ID: ${{ secrets.SPN_GITHUB_CLI_SIGNING_TENANT_ID }}
          DLIB_PATH: ${{ runner.temp }}\acs\bin\x64\Azure.CodeSigning.Dlib.dll
          METADATA_PATH: ${{ runner.temp }}\acs\metadata.json
          TAG_NAME: ${{ inputs.tag_name }}
        run: script/release --local "$TAG_NAME" --platform windows
      - name: Set up MSBuild
        id: setupmsbuild
        uses: microsoft/setup-msbuild@v2.0.0
      - name: Build MSI
        shell: bash
        env:
          MSBUILD_PATH: ${{ steps.setupmsbuild.outputs.msbuildPath }}
        run: |
          for ZIP_FILE in dist/gh_*_windows_*.zip; do
            MSI_NAME="$(basename "$ZIP_FILE" ".zip")"
            MSI_VERSION="$(cut -d_ -f2 <<<"$MSI_NAME" | cut -d- -f1)"
            case "$MSI_NAME" in
            *_386 )
              source_dir="$PWD/dist/windows_windows_386"
              platform="x86"
              ;;
            *_amd64 )
              source_dir="$PWD/dist/windows_windows_amd64_v1"
              platform="x64"
              ;;
            *_arm64 )
              source_dir="$PWD/dist/windows_windows_arm64"
              platform="arm64"
              ;;
            * )
              printf "unsupported architecture: %s\n" "$MSI_NAME" >&2
              exit 1
              ;;
            esac
            "${MSBUILD_PATH}\MSBuild.exe" ./build/windows/gh.wixproj -p:SourceDir="$source_dir" -p:OutputPath="$PWD/dist" -p:OutputName="$MSI_NAME" -p:ProductVersion="${MSI_VERSION#v}" -p:Platform="$platform"
          done
      - name: Sign .msi release binaries
        if: inputs.environment == 'production'
        shell: pwsh
        env:
          AZURE_CLIENT_ID: ${{ secrets.SPN_GITHUB_CLI_SIGNING_CLIENT_ID }}
          AZURE_CLIENT_SECRET: ${{ secrets.SPN_GITHUB_CLI_SIGNING }}
          AZURE_TENANT_ID: ${{ secrets.SPN_GITHUB_CLI_SIGNING_TENANT_ID }}
          DLIB_PATH: ${{ runner.temp }}\acs\bin\x64\Azure.CodeSigning.Dlib.dll
          METADATA_PATH: ${{ runner.temp }}\acs\metadata.json
        run: |
          Get-ChildItem -Path .\dist -Filter *.msi | ForEach-Object {
            .\script\sign.ps1 $_.FullName
          }
      - uses: actions/upload-artifact@v4
        with:
          name: windows
          if-no-files-found: error
          retention-days: 7
          path: |
            dist/*.zip
            dist/*.msi
```
</details>

The purpose of this job is to:
 * Build the executables for Windows, and archive them (`.zip`) using [`GoReleaser`](https://goreleaser.com/)
 * Sign the executables via GoReleaser hook calling [`./script/sign.ps1`](#script-signps1)
 * [Build the MSIs (Installers)](#msi-building) for each architecture
 * Sign the MSIs

### <a id="release">release</a>

<details>

```yml
release:
    runs-on: ubuntu-latest
    needs: [linux, macos, windows]
    environment: ${{ inputs.environment }}
    if: inputs.release
    steps:
      - name: Checkout cli/cli
        uses: actions/checkout@v4
      - name: Merge built artifacts
        uses: actions/download-artifact@v4
      - name: Checkout documentation site
        uses: actions/checkout@v4
        with:
          repository: github/cli.github.com
          path: site
          fetch-depth: 0
          token: ${{ secrets.SITE_DEPLOY_PAT }}
      - name: Update site man pages
        env:
          GIT_COMMITTER_NAME: cli automation
          GIT_AUTHOR_NAME: cli automation
          GIT_COMMITTER_EMAIL: noreply@github.com
          GIT_AUTHOR_EMAIL: noreply@github.com
          TAG_NAME: ${{ inputs.tag_name }}
        run: |
          git -C site rm 'manual/gh*.md' 2>/dev/null || true
          tar -xzvf linux/manual.tar.gz -C site
          git -C site add 'manual/gh*.md'
          sed -i.bak -E "s/(assign version = )\".+\"/\1\"${TAG_NAME#v}\"/" site/index.html
          rm -f site/index.html.bak
          git -C site add index.html
          git -C site diff --quiet --cached || git -C site commit -m "gh ${TAG_NAME#v}"
      - name: Prepare release assets
        env:
          TAG_NAME: ${{ inputs.tag_name }}
        run: |
          shopt -s failglob
          rm -rf dist
          mkdir dist
          mv -v {linux,macos,windows}/gh_* dist/
      - name: Install packaging dependencies
        run: sudo apt-get install -y rpm reprepro
      - name: Set up GPG
        if: inputs.environment == 'production'
        env:
          GPG_PUBKEY: ${{ secrets.GPG_PUBKEY }}
          GPG_KEY: ${{ secrets.GPG_KEY }}
          GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
          GPG_KEYGRIP: ${{ secrets.GPG_KEYGRIP }}
        run: |
          base64 -d <<<"$GPG_PUBKEY" | gpg --import --no-tty --batch --yes
          base64 -d <<<"$GPG_KEY" | gpg --import --no-tty --batch --yes
          echo "allow-preset-passphrase" > ~/.gnupg/gpg-agent.conf
          gpg-connect-agent RELOADAGENT /bye
          /usr/lib/gnupg2/gpg-preset-passphrase --preset "$GPG_KEYGRIP" <<<"$GPG_PASSPHRASE"
      - name: Sign RPMs
        if: inputs.environment == 'production'
        run: |
          cp script/rpmmacros ~/.rpmmacros
          rpmsign --addsign dist/*.rpm
      - name: Attest release artifacts
        if: inputs.environment == 'production'
        uses: actions/attest-build-provenance@520d128f165991a6c774bcb264f323e3d70747f4 # v2.2.0
        with:
          subject-path: "dist/gh_*"
      - name: Run createrepo
        env:
          GPG_SIGN: ${{ inputs.environment == 'production' }}
        run: |
          mkdir -p site/packages/rpm
          cp dist/*.rpm site/packages/rpm/
          ./script/createrepo.sh
          cp -r dist/repodata site/packages/rpm/
          pushd site/packages/rpm
          [ "$GPG_SIGN" = "false" ] || gpg --yes --detach-sign --armor repodata/repomd.xml
          popd
      - name: Run reprepro
        env:
          GPG_SIGN: ${{ inputs.environment == 'production' }}
          # We are no longer adding to the distribution list.
          # All apt distributions should use "stable" according to our install documentation.
          # In the future we will remove legacy distributions listed here.
          RELEASES: "cosmic eoan disco groovy focal stable oldstable testing sid unstable buster bullseye stretch jessie bionic trusty precise xenial hirsute impish kali-rolling"
        run: |
          mkdir -p upload
          [ "$GPG_SIGN" = "true" ] || sed -i.bak '/^SignWith:/d' script/distributions
          for release in $RELEASES; do
            for file in dist/*.deb; do
              reprepro --confdir="+b/script" includedeb "$release" "$file"
            done
          done
          cp -a dists/ pool/ upload/
          mkdir -p site/packages
          cp -a upload/* site/packages/
      - name: Create the release
        env:
          # In non-production environments, the assets will not have been signed
          DO_PUBLISH: ${{ inputs.environment == 'production' }}
          TAG_NAME: ${{ inputs.tag_name }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          shopt -s failglob
          pushd dist
          shasum -a 256 gh_* > checksums.txt
          mv checksums.txt gh_${TAG_NAME#v}_checksums.txt
          popd
          release_args=(
            "$TAG_NAME"
            --title "GitHub CLI ${TAG_NAME#v}"
            --target "$GITHUB_SHA"
            --generate-notes
          )
          if [[ $TAG_NAME == *-* ]]; then
            release_args+=( --prerelease )
          fi
          guard="echo"
          [ "$DO_PUBLISH" = "false" ] || guard=""
          script/label-assets dist/gh_* | xargs $guard gh release create "${release_args[@]}" --
      - name: Publish site
        env:
          DO_PUBLISH: ${{ inputs.environment == 'production' && !contains(inputs.tag_name, '-') }}
          TAG_NAME: ${{ inputs.tag_name }}
          GIT_COMMITTER_NAME: cli automation
          GIT_AUTHOR_NAME: cli automation
          GIT_COMMITTER_EMAIL: noreply@github.com
          GIT_AUTHOR_EMAIL: noreply@github.com
        working-directory: ./site
        run: |
          git add packages
          git commit -m "Add rpm and deb packages for $TAG_NAME"
          if [ "$DO_PUBLISH" = "true" ]; then
            git push
          else
            git log --oneline @{upstream}..
            git diff --name-status @{upstream}..
          fi
      - name: Bump homebrew-core formula
        uses: mislav/bump-homebrew-formula-action@v3
        if: inputs.environment == 'production' && !contains(inputs.tag_name, '-')
        with:
          formula-name: gh
          formula-path: Formula/g/gh.rb
          tag-name: ${{ inputs.tag_name }}
          push-to: williammartin/homebrew-core
        env:
          COMMITTER_TOKEN: ${{ secrets.HOMEBREW_PR_PAT }}
```
</details>

The purpose of this job is to:
 * Aggregate all the previously built artifacts
 * Commit the manual changes to the [CLI site repo](https://github.com/github/cli.github.com/)
 * Use [`./script/createrepo`](#script-createrepo) to create a RedHat package repo metadata (`repomd.xml`)
 * Use GPG to sign the `repomd.xml`
 * Use [rerepro](https://manpages.debian.org/bookworm/reprepro/reprepro.1.en.html) to create a Debian package repository, which signs using the GPG key based on the `SignWith` line in the [`./script/distributions`](#script-distributions) file.
 * Create a GitHub release, labelling assets using [./script/label-assests](#script-labelassets)
 * Commit the package (`.rpm`, `.deb`) repository changes to [CLI site repo](https://github.com/github/cli.github.com/)
 * [Bump the `gh` homebrew formula](#bump-homebrew) using [`mislav/bump-homebrew-formula-action`](https://github.com/mislav/bump-homebrew-formula-action)

### <a id="deepest-dive">Deepest Dive</a>

#### <a id="script-release">./script/release</a>

// TODO: maybe this is part of "releasing"

#### <a id="goreleaser">GoReleaser</a>

#### <a id="notarization">Notarization</a>

#### <a id="script-sign">./script/sign</a>

#### <a id="script-pkgmacos">./script/pkgmacos</a>

#### <a id="script-signps1">./script/sign.ps1</a>

#### <a id="msi-building">MSI Building</a>

#### <a id="script-createrepo">./script/create-repo</a>

#### <a id="script-distributions">./script/distributions</a>

#### <a id="script-labelassets">./script/label-assets</a>

#### <a id="gpg-signing">GPG Signing</a>

#### <a id="bump-homebrew">Bump Homebrew</a>
