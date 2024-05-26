---
description: Wireguard, BGP, Address lists, DNS static, Mangle
---

# Точечная маршрутизация на Mikrotik: Wireguard, BGP, Address lists, DNS static, Mangle

### Введение

Эта статья о том, как настроить точечный роутинг по нужным доменам на роутере Mikrotik с RouterOS 7.x, в частности на модели hAP ac^2. Думаю, большинство понимают для чего нужна точечная маршрутизацию на роутере.

* Есть российские ресурсы, которые блокируют доступ с зарубежных IP
* Сайты определяют вашу локацию по стране, где расположен ваш VPS-сервер. Youtube может показывать вам рекламу, предназначенную для страны в которой расположен ваш сервер.
* Высокая задержка (latency) при открытии внутренних ресурсов, так как создается петля через другую строну для открытия сайта, который расположен недалеко от вас.
* Если у вас не дорогой VPS, то ваш гигабитный канал по оптике может урезаться до 100-200 Мбит.
* Уменьшение нагрузки на VPS сервер.

### Настройка WireGuard в ROS 7

{% hint style="info" %}
Как развернуть сервер Wireguard на своем VPS можно прочитать [здесь](../../../raznoe/docker/dwg.md).
{% endhint %}

Для начала нужно настроить туннель, в который будет направляться трафик для определенных ресурсов. Ниже приведу пример настройки WG:

```bash
/interface wireguard
add comment=vpn listen-port=51820 mtu=1280 name=wireguard1
/interface wireguard peers
add allowed-address=::/0,0.0.0.0/0 endpoint-address=$HOST endpoint-port=51820 interface=wireguard1 preshared-key="$PRESHARED" public-key="$PUBLIC"
/interface list member
add interface=wireguard1 list=LAN
/ip address
add address=$IP/24 comment="vpn network" interface=wireguard1 network=$NETWORK
/ip firewall nat
add action=masquerade chain=srcnat comment="nat for WG" out-interface=wireguard1
```

Проверим что трафик ходит через интерфейс

```bash
/ping ya.ru interface wireguard1
```

Ещё можно глянуть, что значения Rx и Tx ненулевые в WireGuard - Peers

<figure><img src="../../../.gitbook/assets/WireGuard-Peers.png" alt=""><figcaption><p>WireGuard - Peers</p></figcaption></figure>

Теперь приступим к настройке маршрутизации и списков.

Создаем таблицу маршрутизации `wg-mark`, маршруты в туннель для этой таблицы и два правила для маркировки двух списков. В конце сразу правим MSS, так-как с доступом к некоторым сайтам могут возникнуть проблемы.

<pre class="language-bash"><code class="lang-bash">/routing table
add comment="routing tables for vpn" disabled=no fib name=wg-mark
/ip route
<strong>add comment="wg route" disabled=no distance=1 dst-address=0.0.0.0/0 gateway=wireguard1 pref-src="" routing-table=wg-mark scope=30 suppress-hw-offload=no target-scope=10
</strong>add comment="wg subnet" disabled=no distance=1 dst-address=$WGSUBNET/24 gateway=wireguard1 pref-src="" routing-table=main scope=30 suppress-hw-offload=no target-scope=10
/ip firewall mangle
add action=mark-routing chain=prerouting comment="to-wg" dst-address-list=to-wg new-routing-mark=wg-mark passthrough=yes   
add action=mark-routing chain=prerouting comment="vpn-domains" dst-address-list=vpn-domains new-routing-mark=wg-mark passthrough=yes
add action=change-mss chain=forward comment="change mss wireguard1" new-mss=1380 out-interface=wireguard1 passthrough=yes protocol=tcp tcp-flags=syn tcp-mss=1381-65535
</code></pre>

Вместо $WGSUBNET вставляем подсеть туннеля, например 10.2.0.0/24. Теперь по идее все можно работать, можно проверить добавив вручную адрес 2ip.ru в адрес лист.

```bash
/ip firewall address-list
add address=2ip.ru comment=test-2ip list=to-wg
```

Если все настроено правильно сайт должен показать IP адрес VPS сервера. В принципе в этот список можно вручную добавлять нужные вам домены или адреса, но здесь нет поддержки Wildcard. Т.е. не получится перенаправить маршрут сразу всех поддоменов на пример `*.example.com`. Вариант этот использовать можно, но не во сех случаях, поэтому приступим к настройке альтернативных вариантов.

### BGP

Есть два сервиса, которые предоставляют услугу анонса заблокированных IP-адресов и подсетей:

https://antifilter.download/

https://antifilter.network/

Они это делают, чтоб вы могли их заблокировать на роутере. Но для учебных целей их можно направить в туннель.

Первый возник в 2018, второй в 2019 году. Создатели antifilter.network не особо запаривались с неймингом, но запарились с реализацией сервиса.

Оба имеют списки:

* Все IP-адреса, который надо заблокировать
* Их суммаризация для сокращения количества префиксов
* Блокируемые подсети

Чем отличаются?

* У antifilter.download есть список community. Он создаётся из резолвинга доменов, которые добавили пользователи сервиса
* У antifilter.network есть прикольная фича по выбору списков, не прибегая к фильтрации префиксов на роутере
* У antifilter.network есть список с суммаризацией от /32 до /23 маски
* antifilter.network имеет много других списков

У обоих проектов есть чаты, где вы можете задать свой вопрос.

#### Antifilter.download

Настроим получение префиксов из списка subent+ipsum+community от antifilter.download. Эти списки отдаются по умолчанию.

Требуется подставить переменные:

YOUR\_AS - ваша рандомная AS. Число в диапазоне 64512-65543, кроме 65432

IP - ваш внешний IP-адрес для того, чтобы не пересекаться с другими пользователями

INTERFACE - имя VPN интерфейса, через который необходимо пустить трафик

```bash
/routing bgp template
add as=$YOUR_AS disabled=no hold-time=4m input.filter=bgp_in .ignore-as-path-len=yes keepalive-time=1m multihop=yes name=antifilter routing-table=main
/routing bgp connection
add disabled=no hold-time=4m input.filter=bgp_in .ignore-as-path-len=yes keepalive-time=1m local.role=ebgp multihop=yes name=antifilter_bgp remote.address=45.154.73.71/32 .as=65432 router-id=$IP routing-table=main templates=antifilter
/routing filter rule
add chain=bgp_in comment="full list antifilter" disabled=no rule="set gw wireguard1; accept;"
```

Проверить, что префиксы пришли, можно командой

```bash
/routing/bgp/session print
```

Значения prefix-count и messages должны быть не равны нулю.

Либо через winbox\веб-интерфейс. Routing - BGP - Sessions, у поднятой сессии раздел Stats.

<figure><img src="../../../.gitbook/assets/routing-bgp-sessions.png" alt=""><figcaption><p>Routing - BGP - Sessions</p></figcaption></figure>

Делайте трассировку с клиента роутера до необходимого ресурса, маршрут должен проходить через туннель.

Небольшое пояснение перед следующим абзацем. У antifilter.download есть community список, который составляют сами пользователи. Предложить добавить домен можно в чате [Telegram](https://t.me/+zbvV3elo99gzNzhi). Также в BGP существует понятие community, применятся для маркировки части префиксов. В рассматриваемом контексте используется для разделения и использования разных списков. Не запутайтесь.

По умолчанию отдаются вместе subnet, ipsum и community списки.

Чтоб получать только список community, необходимо отредактировать filter rule:

```bash
/routing filter rule
add chain=bgp_in comment="only community list" disabled=no rule="set gw wireguard1; if (bgp-communities includes 65432:500) {accept} else {reject}"
```

Остальные BGP community можно посмотреть на сайте в разделе ЧаВо - "Какие комьюнити используются в BGP-фиде?".

В случае фильтрации будут прилетать все префиксы, но активны будут только из выбранного community.

Посмотреть можно в ip - route

```bash
/ip/route print
```

Dab - active, те, что используются для маршрутизации

DbI - inactive, те, что отфильтровались и для маршрутизации не используются

Если ваш провайдер блокирует BGP, можно сделать маршрут через VPN для связи с сервисом.

```bash
/ip route add dst-address=45.154.73.71/32 gateway=$INTERFACE
```

### Выкачка списков доменов в DNS static, wildcard. Добавлено в ROS 7.5

Некоторые домены в списке community, например те которые стоят за CloudFlare и у них постоянно меняется адрес, добавляются только в [список доменов](https://community.antifilter.download/list/domains.lst).

Благодаря Юрию Ряполову @xen85, [стало известно](https://t.me/itdogcomments/1086) ещё об одном способе добавления IP-адресов доменов в address-list. Большое спасибо ему за это открытие.

В версии 7.5 было добавлена следующая фича

```bash
*) dns - added "address-list" parameter for static DNS entries (CLI only);
*) dns - added "match-subdomain" option for static entries (CLI only);
```

Не обращайте внимание на `CLI only`, в последующих версиях это появилось и в веб-интерфейсе и в winbox. Это реализация по подобию Dnsmasq+ipset.

Добавляем домен в /ip/dns/static

```bash
/ip dns static add name=4pda.to type=FWD forward-to=1.1.1.1 address-list=vpn-domains match-subdomain=yes
```

Теперь IP-адреса домена и его субдоменов при резолвинге будут складываться в address-list vpn-domains. В `type` должен быть `FWD` и обязательно к какому DNS-серверу перенаправлять запрос. Параметр `match-subdomain` добавляет IP-адреса субдоменов указанного домена в `address-list`.

IP-адреса в address-list будет жить, пока живёт запись в `/ip/dns/cache`, а там она живёт, пока не истечёт её TTL.

Тут есть проблема, что обязательно надо куда-то перенаправлять DNS запрос. Нельзя просто без перенаправления складывать в address-list. Ещё нельзя использовать вместе с DoH:

`Currently DoH is not compatible with FWD type static entries, in order to utilize FWD entries, DoH must not be configured.`

Таким образом, если ваш провайдер блокирует/подменяет DNS-запросы можно:

Перенаправлять DNS-запросы в туннель. Нужен настроенный резолвер на IP-адресе гейта VPN-сервера или в его подсети. Использовать отдельный DNS-сервер, который развёрнут рядом с роутером.

Дальше все запросы к зарезолвенным IP-адресам направляются в туннель также с помощью mangle по примеру выше.

Для автоматизации можно использовать скрипт ниже:

```bash
:log info "Starting download domains list script"
:global readfile do={
    :local url        $1
    :local thefile    ""
    :local filesize   ([/tool fetch url=$url as-value output=none]->"downloaded")
    :local maxsize    64512 ; # is the maximum supported readable size of a block from a file
    :local start      0
    :local end        ($maxsize - 1)
    :local partnumber ($filesize / ($maxsize / 1024))
    :local reminder   ($filesize % ($maxsize / 1024))
    :if ($reminder > 0) do={ :set partnumber ($partnumber + 1) }
    :for x from=1 to=$partnumber step=1 do={
         :set thefile ($thefile . ([/tool fetch url=$url http-header-field="Range: bytes=$start-$end" as-value output=user]->"data"))
         :set start   ($start + $maxsize)
         :set end     ($end   + $maxsize)
    }
    :return $thefile
}
 
{
/ip dns static
:local update do={
 :global readfile
 :local data
 :do {
 :set data [$readfile $url]
 } on-error={}
 :log error [:len $data]
 :if ([:len $data] > 0) do={
   :log info "Starting import of domains-list: $listname"
   :log info "Deleting all Dynamic entries in domains-list: $listname"
   # remove the current list completely
   :foreach i in=[/ip dns static find where comment="vpn-domains"] do={
      /ip dns static remove $i
   }
   :local n 0; # counter
   :log info "Imported file length $[:len $data] bytes"
   :while ([:len $data]!=0) do={
     :do { 
       :local line [:pick $data 0 [:find $data "\n"]];
       :if ($line~"^.+\\.[a-z.]{2,7}") do={
        :set n ($n + 1)
        /ip dns static add address-list=$listname disabled=no forward-to=$dns match-subdomain=yes name=$line comment="vpn-domains" regexp="" ttl=1d type=FWD
       }; # if domain present
     } on-error={:log error $error}
   :set data [:pick $data ([:find $data "\n"]+1) [:len $data]]; # removes the just added domain from the data array
   }; # while
   :log info "Completed importing $listname added/replacing $n lines."
 } else={
   :log error "Failed to download domain list, no changes made."
 }
}; # do
 
$update url=("https://" . "community.antifilter.download/list/domains.lst") delimiter=("\n") listname=vpn-domains dns=1.0.0.1
}
```

Сразу запустим Run Script.

Проверить, что адреса добавляются можно в IP - DNS - Static, при запросе домена из это списка он будет добавляться в Firewall - Address Lists.

Добавим в крон System - Scheduler - Add new.

* Policy такие же, как у скрипта
* Interval - раз во сколько времени. Например, раз в 23 часа: 23:00:00.
* On Event - имя скрипта, которое задали при создании скрипта.

Одной командой:

```bash
/system scheduler
add interval=23h name=ScheduleVpnDomainList on-event=VpnDomainList policy=read,write,test
```

Теперь скрипт будет запускаться раз в 23 часа и выгружать список доменов.

{% hint style="info" %}
За основу инструкции была взята статья на [Хабре](https://habr.com/ru/articles/775110/).
{% endhint %}
