# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :otus => {
        :box_name => "centos/7",
        :box_version => "2004.01",
        :ip_addr => '192.168.11.101',
  },
}

Vagrant.configure("2") do |config|
config.vm.synced_folder ".", "/vagrant", disabled: true
  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            	  vb.customize ["modifyvm", :id, "--memory", "1024"]
		  end
#          box.vm.provision "shell", path: "setup_zfs.sh"
 	  box.vm.provision "shell", inline: <<-SHELL
	      mkdir -p ~root/.ssh
              cp ~vagrant/.ssh/auth* ~root/.ssh
              yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils gcc
              cd /root/
	      wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.14.1-1.el7_4.ngx.src.rpm
	      rpm -i nginx-1.14.1-1.el7_4.ngx.src.rpm
	      wget --no-check-certificate https://www.openssl.org/source/openssl-1.1.1o.tar.gz
	      tar -xvf openssl-1.1.1o.tar.gz
	      yum-builddep -y rpmbuild/SPECS/nginx.spec
#	      sed -i 's/--with-debug/--with-debug --with-openssl=\/root\/openssl-1.1.1o/g' rpmbuild/SPECS/nginx.spec
	      sed -i '|--with-debug|i --with-openssl=\/root\/openssl-1.1.1o \\\' rpmbuild/SPECS/nginx.spec
              rpmbuild -bb rpmbuild/SPECS/nginx.spec
	      yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
	      systemctl start nginx
	      mkdir /usr/share/nginx/html/repo
	      cp rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm /usr/share/nginx/html/repo/
	      wget https://repo.percona.com/yum/release/7/RPMS/noarch/percona-release-0.1-6.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm
	      createrepo /usr/share/nginx/html/repo/
	      sed -i 's/index  index.html index.htm;/index  index.html index.htm;\nautoindex on;/g' /etc/nginx/conf.d/default.conf
#	      sed -i '/index  index.html index.htm;/a autoindex on;' /etc/nginx/conf.d/default.conf
#	      nginx -t
	      nginx -s reload
cat >> /etc/yum.repos.d/otus.repo << ERC
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
ERC
	      yum install -y docker
	      systemctl enable docker
	      systemctl start docker
	      docker run -d -p 8080:80 jimidini/otus-linux-day07:nginx
 	  SHELL
      end
  end
end

