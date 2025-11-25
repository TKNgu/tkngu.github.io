+++
date = '2025-11-08T14:21:22+07:00'
draft = true
title = 'BuildDebianOS'
+++
## Hướng dẫn nhanh cách tạo một bản hệ điều hành Ubuntu tùy chỉnh.
Khi cần tạo một hệ điều hành Ubuntu tối giản với các gói phần mềm cần thiết
có thể khởi động từ mạng hoặc qua usb.

## Live build
Công cụ cho phép tạo khung hệ điều hành dựa trên Debian, Ubuntu.
Cho phép thêm các gói phần mềm tùy chỉnh từ kho phần mềm APT.
Đóng gói thành một hệ điều hành đầy đủ có thể khởi động được.

## Chuẩn bị môi trường
sudo apt update
sudo apt install live-build debootstrap squashfs-tools xorriso -y

# Tạo thư mục để build
mkdir MyUbuntu
cd MyUbuntu
lb config \
  --distribution noble \
  --architecture amd64 \
  --linux-flavours amd64 \
  --archive-areas "main restricted" \
  --bootstrap-flavour minimal \

## Thêm các gói sẽ cài 
echo "vim sshfs openssh-server openssh-client net-tools" >> config/package-lists/custom.list.chroot
echo "clonezilla" >> config/package-lists/config.list.chroot

## Build OS
sudo lb build


## Clear Build
sudo lb clean --all
