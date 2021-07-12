#
# This is free software, licensed under the MIT License.
# OpenWRT源 coolsnowwolf/lede
# Description: Build OpenWrt using GitHub Actions
#

name: Build OpenWrt

on:
  repository_dispatch:
  workflow_dispatch:
  schedule:
    - cron: '20 */1 * * *'

env:
  REPO_URL: https://github.com/coolsnowwolf/lede
  REPO_URLs: https://gitlab.com/sarahemersonzrex/sync-quilt.git/
  REPO_BRANCH: master
  FEEDS_CONF: feeds.conf.default
  CONFIG_FILE: newifi-y/.config
  DIY_P1_SH: newifi-y/script.sh
  DIY_P2_SH: x86/part2.sh

  UPLOAD_BIN_DIR: true
  UPLOAD_FIRMWARE: true
  UPLOAD_RELEASE: true
  TZ: Asia/Shanghai

jobs:
  build:
    runs-on: ubuntu-20.04
    timeout-minutes: 359
    steps:
    - name: Checkout
      uses: actions/checkout@main

    - name: Initialization environment
      env:
        DEBIAN_FRONTEND: noninteractive
      run: |
        sudo rm -rf /etc/apt/sources.list.d/* /usr/share/dotnet /usr/local/lib/android /opt/ghc
        sudo -E apt-get -qq update
        sudo -E apt-get -qq install build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch python3 python2.7 unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint device-tree-compiler g++-multilib antlr3 gperf wget curl swig rsync
        sudo -E apt-get -qq autoremove --purge
        sudo -E apt-get -qq clean
        sudo timedatectl set-timezone "$TZ"
        sudo mkdir -p /workdir
        sudo chown $USER:$GROUPS /workdir

    - name: Clone source code
      working-directory: /workdir
      run: |
        df -hT $PWD
        git clone $REPO_URL -b $REPO_BRANCH openwrt
        ls
        ln -sf /workdir/openwrt $GITHUB_WORKSPACE/openwrt

    - name: Load custom feeds
      run: |
        [ -e $FEEDS_CONF ] && mv $FEEDS_CONF openwrt/feeds.conf.default
        chmod +x $DIY_P1_SH
        cd openwrt
        $GITHUB_WORKSPACE/$DIY_P1_SH


    - name: Update feeds
      run: cd openwrt && ./scripts/feeds update -a

    - name: Install feeds
      run: cd openwrt && ./scripts/feeds install -a
      
    - name: Re Update feeds
      run: cd openwrt && ./scripts/feeds update -a

    - name: Re Install feeds
      run: cd openwrt && ./scripts/feeds install -a

    - name: Load custom configuration
      run: |
        [ -e files ] && mv files openwrt/files
        [ -e $CONFIG_FILE ] && mv $CONFIG_FILE openwrt/.config     
        cd openwrt

        
    - name: Download package
      id: package
      run: |
        cd openwrt
        git clone $REPO_URLs
        make defconfig
        make download -j8
        #find dl -size -124c -exec ls -l {} \;
        find dl -size -124c -exec rm -f {} \;

    - name: Compile the firmware
      id: compile
      run: |
        cd openwrt
        cd sync-quilt
        ls
        echo -e "$(nproc) thread compile"
        chmod +x relocatable-patch.sh && ./relocatable-patch.sh
        cd ..
        #make -j$(nproc) || make -j1 || make -j1 V=s
        echo "::set-output name=status::success"
        #grep '^CONFIG_TARGET.*DEVICE.*=y' .config | sed -r 's/.*DEVICE_(.*)=y/\1/' > DEVICE_NAME
        #[ -s DEVICE_NAME ] && echo "DEVICE_NAME=_$(cat DEVICE_NAME)" >> $GITHUB_ENV
        echo "FILE_DATE=_$(date +"%Y%m%d%H%M")" >> $GITHUB_ENV


    - name: Check space usage
      if: (!cancelled())
      run: df -hT

    - name: Organize files
      id: organize
      if: env.UPLOAD_FIRMWARE == 'true' && !cancelled()
      run: |
        #cd openwrt/bin #/targets/*/*
        rm -rf packages
        echo "FIRMWARE=$PWD" >> $GITHUB_ENV
        echo "::set-output name=status::success"

    - name: Generate release tag
      id: tag
      if: env.UPLOAD_RELEASE == 'true' && !cancelled()
      run: |
        echo "::set-output name=release_tag::$(date +"%Y.%m.%d-%H%M"-'lenovo_newifi-y1')"
        touch release.txt       
        echo "::set-output name=status::success"

    - name: Upload firmware to release
      uses: softprops/action-gh-release@v1
      if: steps.tag.outputs.status == 'success' && !cancelled()
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ steps.tag.outputs.release_tag }}
        body_path: release.txt
        files: ${{ env.FIRMWARE }}/*
 
    - name: Delete workflow runs
      uses: GitRML/delete-workflow-runs@main
      with:
        retain_days: 1
        keep_minimum_runs: 3

    - name: Remove old Releases
      uses: dev-drprasad/delete-older-releases@v0.1.0
      if: env.UPLOAD_RELEASE == 'true' && !cancelled()
      with:
        keep_latest: 3
        delete_tags: true
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
