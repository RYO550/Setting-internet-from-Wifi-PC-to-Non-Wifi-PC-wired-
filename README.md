# Setting-internet-from-Wifi-PC-to-Non-Wifi-PC-wired-

背景
ロボット制御PC(ubuntu)で開発していた際、PCIeデバイスを抜き差ししていたら、突然Wifiが接続できなくなりました。
特定のネットワークに接続できないなどのレベルではなく、物理的なWifiモジュールが見つからなくなりました（PCIeデバイス抜き差しにせい？）。
そこで、ネットワークに接続できている他のデバイス（今回はラズパイ4）を介して有線で当該PCにインターネット接続する方法の成功例を説明します。

 
1. Raspberry Piの設定
a. eth0に固定IPアドレスを設定
/etc/dhcpcd.confファイルを編集して、Raspberry Piの有線インターフェース（eth0）に固定IPアドレスを設定します。

ターミナルを開き、以下のコマンドで/etc/dhcpcd.confファイルを編集します。
sudo nano /etc/dhcpcd.conf
ファイルの最後に次の設定を追加します：
interface eth0
static ip_address=192.168.2.1/24
nohook wpa_supplicant
Ctrl + Oで保存し、Enterを押してからCtrl + Xでファイルを閉じます。

設定の説明:
interface eth0: この行は、イーサネットインターフェースeth0に対する設定であることを指定しています。
static ip_address=192.168.2.1/24: eth0に固定IPアドレスとして192.168.2.1を設定します（サブネットマスクは255.255.255.0）。
nohook wpa_supplicant: これは、Wi-Fi設定がイーサネットに干渉しないようにするためのオプションです。

設定を有効にするためにdhcpcdを再起動します。
sudo systemctl restart dhcpcd

b. dnsmasqのインストールと設定
dnsmasqをインストールして、DHCPサーバーとしてRaspberry PiがWindows PCにIPアドレスを配布できるようにします。
sudo apt install dnsmasq
/etc/dnsmasq.confを編集して、DHCP設定を追加します。
sudo nano /etc/dnsmasq.conf
追加する設定内容は以下の通りです：
interface=eth0
dhcp-range=192.168.2.2,192.168.2.20,255.255.255.0,24h
server=8.8.8.8
server=8.8.4.4

interface=eth0: 有線接続のeth0インターフェースを指定します。
dhcp-range=192.168.2.2,192.168.2.20,255.255.255.0,24h: DHCPで配布するIPアドレスの範囲を指定（192.168.2.2から192.168.2.20までの範囲）。
server=8.8.8.8, server=8.8.4.4: GoogleのDNSサーバーを指定。

dnsmasqを再起動して設定を反映させます。
sudo systemctl restart dnsmasq

c. NAT（ネットワークアドレス変換）設定の追加
Raspberry PiがWi-Fiでインターネットに接続し、Windows PCにその接続を共有できるようにするため、NATを設定します。

NATのルールを追加して、インターネット接続をwlan0インターフェースからeth0に共有します。
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT

MASQUERADE: Wi-Fiインターフェースwlan0を通じてインターネットに接続します。
FORWARD: 有線インターフェースeth0からWi-Fiへの転送を許可します。

NAT設定を永続化して、再起動後も設定が保持されるようにします。
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"

/etc/rc.localを編集して、システム起動時にNAT設定を自動的に適用させます。
sudo nano /etc/rc.local
exit 0の前に以下の行を追加します：
iptables-restore < /etc/iptables.ipv4.nat
ファイルを保存して閉じたら、再起動後もNAT設定が適用されます。

以上で基本的には完了するはずです。
できない場合はChatgptに聞いてください。

また、ラズパイを再起動した場合は下記をそれぞれ実行してください。

有線によるインターネット共有の有効化
sudo systemctl restart dnsmasq
IP転送の有効化
sudo sysctl net.ipv4.ip_forward=1

