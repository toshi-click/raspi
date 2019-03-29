# -*- mode: ruby -*-
# vi: set ft=ruby :

# coding: utf-8
VAGRANTFILE_API_VERSION = "2"

# 任意の場所で失敗させた時にエラーメッセージ出すために定義
def fail_with_message(msg)
  fail Vagrant::Errors::VagrantError.new, msg
end

# vagrant up時にプラグインを自動でインストールしたい場合に使用する。でも勝手に入るのは怖いから使ってない。
def install_plugin(plugin)
  system "vagrant plugin install #{plugin}" unless Vagrant.has_plugin? plugin
end

# Windowsの場合はvboxfs使う可能性があるので、共有用ソフトを新しくするプラグインが入っているか確認する
if Vagrant::Util::Platform.windows? and !Vagrant.has_plugin? 'vagrant-vbguest'
  fail_with_message "vagrant-vbguest missing, please install the plugin with this command:\nvagrant plugin install vagrant-vbguest"
end

# ファイルシステムのマウント用プラグイン入れる
install_plugin "vagrant-vbguest"

# VMのディスク容量を設定できるようにするプラグイン入れる
install_plugin "vagrant-disksize"

# 初回構築中に1度VM再起動が必要なのでそれを可能にするプラグイン入れる
install_plugin "vagrant-reload"

install_plugin "vagrant-bindfs"

# hostsに自動登録してくれるプラグイン入れる
install_plugin "vagrant-hostmanager"
# windows でだけ必要なプラグイン
if Vagrant::Util::Platform.windows? and !Vagrant.has_plugin? 'vagrant-winnfsd'
  install_plugin "vagrant-winnfsd"
end

Vagrant.configure("2") do |config|
  # ubuntu 18.04
  config.vm.box = "ubuntu/bionic64"

  if Vagrant.has_plugin? 'vagrant-disksize'
    config.disksize.size = '30GB'
  end

  # NFS共有を動かすためにプライベートネットワークが必要
  config.vm.network "private_network", ip: "192.168.254.105"
  # ポートフォワードの設定(必要な時だけコメントアウトを解除する運用です。ホストのポートが競合すると他のVMが立ち上げられなくなるので)
  # `host:` で指定しているポートへ通信が来たら `guest:` に設定されているポートにポートフォワードします。
  # PC本体が `host:` 、VMが `guest:` です。
  if Vagrant::Util::Platform.windows? or Vagrant::Util::Platform.wsl?
    # DockerのAPIを実行したいのでポート開放
    config.vm.network :forwarded_port, host: 2370, guest: 2375
    # config.vm.network :forwarded_port, host: 25, guest: 25
    # config.vm.network :forwarded_port, host: 110, guest: 110
  else
    # config.vm.network :forwarded_port, host: 25, guest: 25
    # config.vm.network :forwarded_port, host: 110, guest: 110
  end

  # 構築に必要なリソース
  # VirtualboxのGUI上で見える名前など設定
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 2
    v.name = "vm.raspi.test"
  end

  # ホスト名登録すると自動でhostsに登録されるようにできるので設定する
  # proxyに設定する正しいFQDNにすること。
  config.vm.hostname = "vm.raspi.test"

  # ホスト名を自動的に入れる。ホスト機とVM内に。
  # vagrant up時とvagrant destroy時に動作する
  if Vagrant.has_plugin?('vagrant-hostmanager')
    redirects = "api.vm.raspi.test"
    # デフォルトではvagrant hostmanagerを実行しないと動作しないが、以下の行を追加しておけばvagrant up時に自動的にhosts設定を行ってくれる。
    config.hostmanager.enabled = true
    # 以下の行を追加すると、ホストOSのhostsへも追加してくれる。
    config.hostmanager.manage_host = true
    # 以下の行を追加すると、ゲストOSのhostsへも追加してくれる。
    config.hostmanager.manage_guest = true
    # config.vm.hostname以外の名前付けがしたい場合に設定する。
    config.hostmanager.aliases = redirects
  end

  if Vagrant::Util::Platform.wsl?
    config.vm.synced_folder ".", "/app/raspi", owner: 'vagrant', group: 'vagrant', mount_options: ['dmode=776', 'fmode=775']
  else
    config.bindfs.bindfs_version = "1.12.2"
    config.bindfs.install_bindfs_from_source = true
    config.vm.synced_folder ".", "/vagrant", type: "virtualbox"
    config.vm.synced_folder ".", "/nfs/mount", type: "nfs", :nfs => {mount_options: ['dmode=777', 'fmode=777']}
    # config.bindfs.force_empty_mountpoint = true
    # nfsでマウントしたファイルをさらにマウントし直す。権限および所有者を設定できるようにするため。
    config.bindfs.bind_folder "/nfs/mount", "/app/raspi",
                              :owner => "toshi",
                              :group => "toshi",
                              :'create-as-user' => true,
                              :perms => "u=rw:g=rw:o=r",
                              :'create-with-perms' => "u=rw:g=rw:o=r",
                              :'chown-ignore' => true,
                              :'chgrp-ignore' => true,
                              :'chmod-ignore' => true,
                              o: :nonempty
  end

  config.vm.provision "ansible_local", run: "always" do |ansible|
    ansible.install_mode = "pip"
    ansible.limit = "dev_local"
    ansible.inventory_path = "./provision/inventory/default.yml"
    ansible.playbook = "./provision/vm_init.yml"
  end

  # 一般ユーザーを指定してのマウントを有効にするためにここでリロード
  config.vm.provision :reload


  config.vm.provision "shell", run: "always", inline: <<-SHELL
cat /vagrant/.ansible_vault_pass > /tmp/.ansible_vault_pass
chmod 644 /tmp/.ansible_vault_pass
#systemctl restart networking.service
  SHELL

  config.vm.provision "ansible_local", run: "always" do |ansible|
    ansible.install_mode = "pip"
    ansible.limit = "dev_local"
    ansible.inventory_path = "./provision/inventory/default.yml"
    ansible.playbook = "./provision/base_setting.yml"
    ansible.vault_password_file = "/tmp/.ansible_vault_pass"
  end

  # config.vm.provision "ansible_local", run: "always" do |ansible|
  #   ansible.limit = "dev_local"
  #   ansible.inventory_path = "./provision/inventory/default.yml"
  #   ansible.playbook = "./provision/02_dev_service.yml"
  #   ansible.vault_password_file = "/tmp/.ansible_vault_pass"
  # end

  # ansibleでVMの基礎設定行う
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo "Start of the command always executed at the end"
  SHELL

end
