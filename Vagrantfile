# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Используем образ Oracle Linux 9
  config.vm.box = "oraclelinux/9"
  config.vm.box_url = "https://oracle.github.io/vagrant-projects/boxes/oraclelinux/9.json"

  # Настройки виртуальной машины
  config.vm.hostname = "packages"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 8192 # 8GB RAM
    vb.cpus = 2
    # Включаем Nested Virtualization
    vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
  end

  # Провижнинг через shell для выполнения всех шагов
  config.vm.provision "shell", inline: <<-SHELL
    # Настраиваем DNS-серверы 8.8.8.8 и 8.8.4.4
    echo "nameserver 8.8.8.8" > /etc/resolv.conf
    echo "nameserver 8.8.4.4" >> /etc/resolv.conf
    # Защищаем resolv.conf от перезаписи
    chattr +i /etc/resolv.conf

    # Обновляем систему
    dnf update -y

    # Устанавливаем необходимые пакеты
    dnf install -y wget rpmdevtools rpm-build createrepo dnf-utils cmake gcc git nano policycoreutils-python-utils

    # Создаем директорию для работы с RPM
    mkdir -p /root/rpm
    cd /root/rpm

    # Загружаем SRPM пакет Nginx
    dnf install -y epel-release
    dnf install -y --enablerepo=epel nginx
    dnf download --source nginx

    # Устанавливаем SRPM и зависимости для сборки
    rpm -Uvh nginx*.src.rpm
    dnf builddep -y nginx

    # Скачиваем и собираем модуль ngx_brotli
    cd /root
    git clone --recurse-submodules -j8 https://github.com/google/ngx_brotli
    cd ngx_brotli/deps/brotli
    mkdir out && cd out
    cmake -DCMAKE_BUILD_TYPE=Release \
          -DBUILD_SHARED_LIBS=OFF \
          -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" \
          -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" \
          -DCMAKE_INSTALL_PREFIX=./installed ..
    cmake --build . --config Release -j 2 --target brotlienc
    cd /root

    # Правим spec-файл для добавления модуля ngx_brotli
    sed -i '/--with-stream_ssl_preread_module/a --add-module=/root/ngx_brotli \\' /root/rpmbuild/SPECS/nginx.spec

    # Собираем RPM-пакет
    cd /root/rpmbuild/SPECS
    rpmbuild -ba nginx.spec -D 'debug_package %{nil}'

    # Копируем RPM-пакеты в одну директорию
    cp /root/rpmbuild/RPMS/x86_64/*.rpm /root/rpmbuild/RPMS/

    # Устанавливаем собранный Nginx
    dnf localinstall -y /root/rpmbuild/RPMS/nginx-*.rpm
    systemctl enable nginx
    systemctl start nginx

    # Создаем директорию для репозитория
    mkdir -p /usr/share/nginx/html/repo
    chown -R nginx:nginx /usr/share/nginx/html/repo
    chmod -R 755 /usr/share/nginx/html/repo
    chcon -R -t httpd_sys_content_t /usr/share/nginx/html/repo/

    # Копируем RPM-пакеты в репозиторий
    cp /root/rpmbuild/RPMS/*.rpm /usr/share/nginx/html/repo/

    # Инициализируем репозиторий
    createrepo /usr/share/nginx/html/repo/

    # Переприменяем права и SELinux контекст после createrepo
    chown -R nginx:nginx /usr/share/nginx/html/repo
    chmod -R 755 /usr/share/nginx/html/repo
    chcon -R -t httpd_sys_content_t /usr/share/nginx/html/repo/

    # Обеспечиваем права на лог-директорию Nginx
    mkdir -p /var/log/nginx
    chown -R nginx:nginx /var/log/nginx
    chmod -R 755 /var/log/nginx
    chcon -R -t httpd_log_t /var/log/nginx/

    # Настраиваем Nginx в главном конфигурационном файле
    cat > /etc/nginx/nginx.conf << EOF
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format  main  '\$remote_addr - \$remote_user [\$time_local] "\$request" '
                      '\$status \$body_bytes_sent "\$http_referer" '
                      '"\$http_user_agent" "\$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    error_log   /var/log/nginx/repo_error.log debug;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80;
        server_name  localhost;

        location /repo/ {
            root /usr/share/nginx/html;
            index index.html index.htm;
            autoindex on;
            autoindex_exact_size off;
            autoindex_localtime on;
        }
    }
}
EOF

    # Проверяем синтаксис и перезапускаем Nginx
    nginx -t
    nginx -s reload

    # Проверяем, под каким пользователем работает Nginx
    echo "Nginx user: $(ps -u -C nginx | grep nginx | awk '{print $1}')"

    # Проверяем SELinux denials
    echo "Checking SELinux denials:"
    audit2allow -a

    # Создаем директорию для репозиториев, если не существует
    mkdir -p /etc/yum.repos.d

    # Создаем otus.repo с помощью tee
    echo "[otus]" | tee /etc/yum.repos.d/otus.repo
    echo "name=otus-linux" | tee -a /etc/yum.repos.d/otus.repo
    echo "baseurl=http://localhost/repo" | tee -a /etc/yum.repos.d/otus.repo
    echo "gpgcheck=0" | tee -a /etc/yum.repos.d/otus.repo
    echo "enabled=1" | tee -a /etc/yum.repos.d/otus.repo

    # Устанавливаем права на otus.repo
    chmod 644 /etc/yum.repos.d/otus.repo

    # Проверяем, что файл создан
    echo "Contents of /etc/yum.repos.d/otus.repo:"
    cat /etc/yum.repos.d/otus.repo

    # Очищаем кэш и обновляем метаданные
    dnf clean all
    dnf repolist
    dnf makecache

    # Проверяем, что репозиторий otus распознан
    echo "Checking otus repo:"
    dnf repolist enabled | grep otus

    # Проверяем содержимое репозитория
    echo "Packages in otus repo:"
    dnf list --repo=otus

    # Временно отключаем SELinux для проверки доступа
    echo "Current SELinux mode: $(getenforce)"
    setenforce 0

    # Добавляем сторонний пакет в репозиторий
    cd /usr/share/nginx/html/repo/
    wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm
    createrepo /usr/share/nginx/html/repo/

    # Переприменяем права и SELinux контекст после добавления нового пакета
    chown -R nginx:nginx /usr/share/nginx/html/repo
    chmod -R 755 /usr/share/nginx/html/repo
    chcon -R -t httpd_sys_content_t /usr/share/nginx/html/repo/

    # Проверяем доступ к репозиторию
    echo "Checking repository access:"
    curl -I http://localhost/repo/
    curl http://localhost/repo/

    # Очищаем кэш и обновляем метаданные снова
    dnf clean all
    dnf makecache

    # Проверяем содержимое репозитория снова
    echo "Packages in otus repo after adding percona-release:"
    dnf list --repo=otus

    # Устанавливаем percona-release
    dnf install -y percona-release-latest.noarch

    # Восстанавливаем SELinux в Enforcing
    setenforce 1
    echo "SELinux mode restored: $(getenforce)"

    # Проверяем статус репозитория
    dnf repolist enabled | grep otus
    dnf list | grep otus

    # Проверяем логи Nginx для ошибок
    echo "Nginx error log:"
    cat /var/log/nginx/repo_error.log || echo "No error log found"

    # Проверяем права и SELinux контекст
    echo "Repository directory permissions and SELinux context:"
    ls -ldZ /usr/share/nginx/html/repo/
    ls -lZ /usr/share/nginx/html/repo/

    # Выводим сообщение об успешном завершении
    echo "Репозиторий доступен по адресу http://localhost/repo/"
  SHELL
end