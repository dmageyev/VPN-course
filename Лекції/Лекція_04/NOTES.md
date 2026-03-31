# Конспект — Лекція 04 — Протоколи тунелювання

### 1. Поняття тунелювання та інкапсуляції

**Тунелювання** — це механізм передачі пакетів одного мережевого протоколу всередині пакетів іншого протоколу. Зовнішній протокол виконує роль «обгортки» (carrier), а внутрішній — «пасажира» (passenger).

Ключові компоненти:
- **Протокол-пасажир (passenger protocol)** — вихідний пакет, що інкапсулюється (IPv4, IPv6, Ethernet)
- **Протокол інкапсуляції (encapsulating protocol)** — додає власний заголовок (GRE, L2TP, IPsec)
- **Транспортний протокол (transport protocol)** — забезпечує доставку (IP, UDP, TCP)

Тунелювання охоплює рівні 2, 3 і 4 моделі OSI залежно від протоколу.

### 2. PPTP

**PPTP (Point-to-Point Tunneling Protocol)** розроблено Microsoft у 1996 р. Інкапсулює кадри PPP у пакети IP через протокол GRE. Керуючий канал працює на TCP-порту 1723.

Автентифікація: MS-CHAPv1/v2 (вразливі до атак грубої сили та перехоплення).  
**Статус**: застарілий; не рекомендується до використання через критичні вразливості.

### 3. L2TP

**L2TP (Layer 2 Tunneling Protocol)** — RFC 2661 (L2TPv2), RFC 3931 (L2TPv3). Розроблений спільно Cisco та Microsoft, поєднує переваги PPTP і L2F. Не має власного шифрування — завжди використовується разом з IPsec (L2TP/IPsec).

Порт: UDP 1701 (всередині IPsec). Подвійна інкапсуляція збільшує накладні витрати, проте забезпечує надійний захист.

### 4. GRE

**GRE (Generic Routing Encapsulation)** — RFC 2784. Протокол рівня 3, розроблений Cisco. Інкапсулює будь-який мережевий протокол у IP-пакети. Не забезпечує шифрування чи автентифікацію самостійно — зазвичай комбінується з IPsec (GRE over IPsec або IPsec over GRE).

Застосування: транспортування IPv6 через IPv4, MPLS, поєднання з динамічними протоколами маршрутизації (OSPF, EIGRP) у VPN-тунелях.

### 5. SSTP

**SSTP (Secure Socket Tunneling Protocol)** — розроблений Microsoft (Windows Vista SP1+). Інкапсулює PPP-трафік у HTTPS (TCP-порт 443). Ключова перевага — проходить через більшість міжмережевих екранів та проксі.

Підтримка: переважно екосистема Windows; обмежена підтримка на Linux/macOS.

### 6. IKEv2

**IKEv2 (Internet Key Exchange version 2)** — RFC 7296. Розроблений Microsoft і Cisco. Використовується для встановлення IPsec SA (Security Association). Протокол UDP 500/4500.

Особливість **MOBIKE (RFC 4555)**: підтримка зміни IP-адреси без розриву тунелю — ідеальний вибір для мобільних пристроїв. Швидке відновлення з'єднання після переривання.

### 7. Порівняльна таблиця протоколів

| Протокол   | Рівень OSI | Шифрування | Порт         | Швидкість | Безпека | Платформи        |
|------------|-----------|------------|-------------|-----------|---------|-----------------|
| PPTP       | 2         | MPPE (40/128 біт) | TCP 1723 | Висока  | Низька  | Широка          |
| L2TP/IPsec | 2         | IPsec (AES) | UDP 1701/500/4500 | Середня | Висока | Широка         |
| GRE        | 3         | Немає (з IPsec — є) | IP 47 | Висока | Залежить від IPsec | Cisco+       |
| SSTP       | 2         | SSL/TLS    | TCP 443     | Середня   | Висока  | Windows+         |
| IKEv2/IPsec| 3         | IPsec (AES) | UDP 500/4500 | Висока  | Висока  | Широка (мобільні)|

### 8. GRE keepalive та DMVPN

GRE keepalive — механізм виявлення розриву тунелю. GRE endpoint надсилає спеціальний keepalive пакет, інкапсульований у GRE пакет з src та dst IP зміненими місцями. Якщо маршрутизатор-партнер отримає цей пакет, він обробить його як звичайний GRE пакет і надішле назад. Якщо протягом заданого часу немає відповіді N разів — тунель оголошується DOWN. Без keepalive GRE тунель лишається технічно піднятим навіть якщо партнер недоступний.

DMVPN (Dynamic Multipoint VPN) — технологія Cisco для масштабування VPN-мереж без необхідності конфігурувати тунель між кожною парою spoke. Три ключові компоненти: mGRE (Multipoint GRE) — один GRE-інтерфейс може підтримувати з'єднання з кількома партнерами одночасно (dynamic tunnel endpoints); NHRP (Next Hop Resolution Protocol, RFC 2332) — протокол «адресної книги»: spoke реєструє свою зовнішню IP-адресу у hub (NHS — Next Hop Server), і може запитати у hub адресу будь-якого іншого spoke; IPsec — шифрування трафіку в тунелях.

Три фази DMVPN: Фаза 1 — hub-and-spoke, весь трафік через hub; Фаза 2 — spoke-to-spoke прямі тунелі на вимогу (spoke запитує NHRP у hub і встановлює прямий тунель); Фаза 3 — оптимізована маршрутизація, summarization та shortcuts через NHRP redirect.

### 9. L2TPv3 та Ethernet Pseudowires

L2TPv3 (RFC 3931) — значно розширена версія L2TP, призначена для транспортування Layer 2 кадрів через IP мережу. Концепція Pseudowire (PW): емуляція point-to-point L2 з'єднання (ніби між двома маршрутизаторами є пряма фізична лінія) через пакетну IP/MPLS мережу.

Типи Pseudowire у L2TPv3: Ethernet (тип 0x0005) — пряма передача Ethernet кадрів, VLAN (тип 0x0004) — збереження VLAN тегів, Frame Relay (тип 0x0001), ATM (тип 0x0002, 0x0003), HDLC (тип 0x0006). L2TPv3 не залежить від MPLS і може використовувати звичайний IP/UDP транспорт.

Застосування: оператори телекомунікацій використовують L2TPv3 для надання L2 VPN послуг клієнтам через IP backbone. «Прозора» передача Ethernet між двома офісами клієнта — трафік між офісами виглядає як в одному LAN (Layer 2 доступність).

### 10. IKEv2 payload types та структура повідомлень

IKEv2 повідомлення складається з фіксованого IKE заголовка та ланцюжка payload-ів. Кожен payload має Generic Payload Header: Next Payload (8 біт), Critical Flag (1 біт), Reserved (7 біт), Payload Length (16 біт).

Ключові payload типи: SA (Security Association, тип 33) — пропозиції наборів алгоритмів (Proposals → Transforms → Attribute), KE (Key Exchange, тип 34) — DH public value та DH group number, Nonce (тип 40) — випадкове число для захисту від replay та деривації ключів, IDi/IDr (тип 35/36) — ідентифікатор ініціатора або responder (IP address, FQDN, e-mail, Distinguished Name), AUTH (тип 39) — підпис або HMAC для аутентифікації (RSA Signature, ECDSA, PSK, EAP), CERT (тип 37) — X.509 сертифікат, CERTREQ (тип 38) — запит на сертифікат певного CA, TSi/TSr (тип 44/45) — Traffic Selectors (IP ranges та port ranges), SK (тип 46) — зашифрований та автентифікований payload-контейнер для захисту IKE_AUTH та інших захищених обмінів, CP (тип 47) — Configuration Payload для передачі внутрішньої IP-адреси, DNS серверів тощо.

### 11. EAP автентифікація в IKEv2

EAP (Extensible Authentication Protocol) у IKEv2 дозволяє використовувати зовнішні методи аутентифікації, не визначені безпосередньо в IKEv2. Це критично для інтеграції з корпоративними системами (Active Directory, RADIUS).

Процес EAP в IKEv2: Ініціатор автентифікує себе відправкою AUTH payload з особливим значенням «I will use EAP»; Responder відповідає своїм AUTH (на основі сертифіката) та надсилає EAP Request-Identity; Ініціатор надсилає EAP Response-Identity; Далі відбувається власне EAP автентифікація (серія EAP Request/Response, тип залежить від методу); IKEv2 тунелює EAP повідомлення в payload SK.

Поширені EAP методи у IKEv2: EAP-TLS (тип 13) — клієнт автентифікується власним X.509 сертифікатом у TLS handshake; EAP-PEAP — тунель TLS захищає внутрішній EAP метод; EAP-MSCHAPv2 (тип 26) — парольна аутентифікація (ім'я + пароль) за протоколом MSCHAPv2; EAP-TTLS — схожий на PEAP але більш гнучкий щодо внутрішніх методів.

RADIUS інтеграція: IKEv2 responder (VPN-шлюз) виступає RADIUS client. Отримавши EAP дані від ініціатора, пересилає їх на RADIUS сервер (FreeRADIUS, Cisco ISE, Microsoft NPS). RADIUS перевіряє облікові дані проти Active Directory або LDAP та повертає Access-Accept або Access-Reject. Цей підхід дозволяє централізоване управління VPN-аутентифікацією.

### 12. Атаки на VPN та методи захисту

Tunnel hijacking: зловмисник у спільній мережі намагається вставити підроблені GRE або ESP пакети у існуючий тунель. Захист: IPsec автентифікація (ICV/HMAC) виявляє підроблені пакети; Sequence Number захищає від replay.

Брутфорс IKE Aggressive Mode: якщо VPN-шлюз прийнятний PSK та Aggressive Mode — зловмисник перехоплює пакети та офлайн атакує HMAC. Захист: не використовувати Aggressive Mode з PSK; використовувати IKEv2 або Strong PSK.

VPN over TLS fingerprinting та блокування: DPI (Deep Packet Inspection) системи можуть виявити характерні ознаки OpenVPN, WireGuard трафіку та заблокувати його. Захист: obfuscation (obfs4proxy для OpenVPN, obfsucation для WireGuard), маскування під HTTPS через TCP 443.

CPU exhaustion (IKE flooding): зловмисник надсилає тисячі IKE_SA_INIT запитів, навантажуючи CPU VPN-шлюзу на DH-обчислення. Захист: IKEv2 cookies (COOKIE notify payload) — перед виконанням DH, шлюз вимагає від ініціатора echo COOKIE; це підтверджує валідність src IP і захищає від IP-spoofed floods.

SSL-Stripping та MITM для SSL VPN: атаки на HTTPS компонент SSL VPN. Захист: HSTS, Certificate Pinning у VPN клієнтах, OCSP/CRL перевірка сертифікатів.

### 13. QUIC та MASQUE — майбутнє тунелювання

QUIC (RFC 9000) — сучасний транспортний протокол, розроблений Google та стандартизований IETF у 2021. Побудований поверх UDP, нативно включає TLS 1.3, підтримує multiplexing та Connection Migration.

MASQUE (Multiplexed Application Substrate over QUIC Encryption, RFC 9298) — стандарт IETF для тунелювання через QUIC. Дозволяє проксіювати UDP та TCP трафік через QUIC з'єднання. Основа для майбутніх VPN-рішень поверх QUIC.

Переваги QUIC-based VPN: 0-RTT відновлення після розриву з'єднання (порівняно з повним TLS handshake у OpenVPN), Connection Migration — зміна IP-адреси без розриву тунелю (краще ніж MOBIKE), нативний multiplexing — кілька логічних потоків в одному UDP підключенні, вбудований TLS 1.3 без окремого шару.

Cloudflare вже використовує MASQUE у своєму Warp клієнті. Apple Private Relay також використовує MASQUE для маршрутизації трафіку. Це підтверджує практичну зрілість технології.

### 14. Детальний аналіз протоколу PPTP

PPTP (Point-to-Point Tunneling Protocol, RFC 2637) розроблено Microsoft у 1996 році як перший широко доступний VPN-протокол. Незважаючи на застарілість, важливо розуміти його архітектуру для розуміння еволюції VPN-протоколів.

Архітектура PPTP: два окремих канали. По-перше, Control Connection на TCP порту 1723 — встановлюється до передачі даних, несе керуючі повідомлення (Start-Control-Connection-Request, Call-Setup, Echo-Request/Reply для keepalive). По-друге, Data Tunnel — GRE версія 1 (enhanced GRE, RFC 2637) інкапсулює PPP кадри у IP. На відміну від стандартного GRE (RFC 2784), PPTP GRE має поля Acknowledgment Number та Key (Contains Call ID).

PPP (Point-to-Point Protocol) є основою: PPTP тунелює PPP кадри, які, у свою чергу, несуть IP-пакети клієнта. PPP надає: LCP (Link Control Protocol) для встановлення з'єднання, NCP (Network Control Protocol) для узгодження IP-параметрів (IP-адреса клієнта), PAP/CHAP або MS-CHAP для аутентифікації.

MS-CHAPv2 (Microsoft Challenge Handshake Authentication Protocol v2) — протокол аутентифікації у PPTP: сервер надсилає 16-байтний challenge; клієнт відповідає NT-hash паролю, обробленим через DES з 8-байтними sub-challeng'ами. Вразливість: NT-hash (MD4 від пароля) можна відновити зі challenge/response через offline brute-force. Сервіс CloudCracker у 2012 році демонстрував зламування за кілька годин.

MPPE (Microsoft Point-to-Point Encryption): шифрування трафіку у PPTP. Використовує RC4 (40 або 128 біт). RC4 є поточним шифром зі складними вразливостями (WEP, RC4 в TLS). Ключ RC4 деривується з NT-hash пароля — тому якщо пароль слабкий, шифрування слабке. Статус: PPTP повністю заборонений NIST та усіма міжнародними стандартами безпеки.

### 15. Детальний аналіз L2TP архітектури

L2TP компонентна архітектура: LAC (L2TP Access Concentrator) — пристрій на стороні клієнта або ISP, що встановлює L2TP тунель. LNS (L2TP Network Server) — пристрій на стороні корпоративної мережі або VPN-сервера, що приймає L2TP тунелі та термінує PPP сесії.

L2TP тунель vs L2TP сесія: один тунель може містити кілька сесій (сесія для кожного PPP-підключення). Tunnel ID ідентифікує тунель між LAC та LNS; Session ID ідентифікує конкретну PPP сесію всередині тунелю.

L2TP Control Messages (Control Connection на UDP 1701): SCCRQ/SCCRP (Start-Control-Connection-Request/Reply) для встановлення тунелю, ICCN/OCRQ (Incoming/Outgoing Call Request) для встановлення сесії, WEN (WAN Error Notify) для звітування про помилки, SLI (Set Link Info) для узгодження PPP параметрів, Hello для keepalive (відсутність відповіді — тунель DOWN).

L2TP/IPsec порядок встановлення з'єднання: Phase 1 — IKE Main Mode або Aggressive Mode → встановлення IKE SA; Phase 2 — Quick Mode → встановлення IPsec SA для захисту L2TP UDP трафіку; Phase 3 — L2TP тунель (SCCRQ/SCCRP); Phase 4 — L2TP сесія (ICCN/OCRQ); Phase 5 — PPP LCP, NCP, автентифікація (MS-CHAPv2 або EAP); Phase 6 — IP призначається, трафік тече.

Подвійна інкапсуляція L2TP/IPsec: зовнішній IP + UDP 4500 (NAT-T) + IPsec ESP + UDP 1701 + L2TP Header + PPP + внутрішній IP + TCP/UDP + дані. Overhead: ~70-90 байтів. Це значно більше ніж WireGuard (~32 байти) або IKEv2 (~50 байтів).

### 16. SSTP детальна архітектура

SSTP (Secure Socket Tunneling Protocol) розроблений Microsoft для Windows Vista SP1. Ключова ідея: тунелювати PPP трафік через HTTPS, що дозволяє обходити firewall та корпоративні проксі, які дозволяють HTTPS на TCP 443.

Процес встановлення SSTP з'єднання: Клієнт встановлює TCP 443 до SSTP сервера; TLS Handshake: клієнт перевіряє сертифікат сервера (повинен бути виданий довіреним CA або сертифікат вказаний вручну); SSTP Capability Request/Response — узгодження параметрів SSTP; PPP Negotiation (LCP, NCP, аутентифікація); IP-адреса призначається, трафік тече.

SSTP пакети: SSTP Data Packet — несе PPP кадри, розбиті на SSTP payload; SSTP Control Packet — контрольні повідомлення (Call Connect Ack, Call Connected, Abort, Disconnect). Всі пакети SSTP передаються як HTTP POST body до кінцевої точки /sra_{BA195980-CD49-458b-9E23-C84EE0ADCD75}/. Цей URL унікальний для SSTP і ідентифікується DPI — тому SSTP легко розпізнається та блокується сучасними DPI системами, незважаючи на використання порту 443.

Обмеження SSTP: офіційно підтримується лише на Windows. SoftEther є open-source реалізацією SSTP-сумісного протоколу. На Linux можна використовувати sstp-client. macOS не має вбудованого SSTP клієнта.

### 17. Вибір протоколу для сучасних розгортань

Перша порада — ніколи не використовуйте PPTP. Незалежно від вимог, є кращі альтернативи.

Для нових корпоративних Site-to-Site розгортань: IKEv2/IPsec є оптимальним вибором. Підтримується апаратно більшістю мережевих пристроїв, стандартизований, висока продуктивність. Альтернатива: WireGuard або GRE/IPsec для Cisco-середовищ.

Для мобільного Remote Access: IKEv2 з MOBIKE для плавного переключення між Wi-Fi та мобільним Інтернетом. OpenVPN як крос-платформенна альтернатива. WireGuard для нових розгортань де важлива продуктивність.

Для обходу рестриктивних firewall: OpenVPN over TCP 443 або SSTP. Обфускація (obfs4proxy, Shadowsocks) для обходу DPI. QUIC-based рішення (Cloudflare WARP, MASQUE) для майбутнього.

Для legacy сумісності: L2TP/IPsec для старих пристроїв Windows без підтримки IKEv2. GRE для Cisco IOS роутерів зі старою прошивкою.

Таблиця рекомендацій:
| Сценарій | Рекомендований протокол | Альтернатива |
|---|---|---|
| Новий Site-to-Site | IKEv2/IPsec | WireGuard |
| Мобільний Remote Access | IKEv2 (MOBIKE) | OpenVPN |
| Обхід firewall | OpenVPN TCP 443 | SSTP |
| Висока продуктивність | WireGuard | IKEv2/IPsec з AES-NI |
| Legacy пристрої | L2TP/IPsec | GRE/IPsec |
| IoT/вбудовані | WireGuard | IPsec ESP |

### 18. Підсумок та ключові концепції

Протоколи тунелювання є фундаментальним шаром будь-якої VPN-архітектури. Їх розуміння дозволяє свідомо обирати рішення, а не покладатись на маркетингові твердження.

Ключові висновки: по-перше, всі протоколи тунелювання реалізують одну і ту саму базову ідею — інкапсуляцію — але відрізняються рівнем OSI, підходом до безпеки, overhead та сумісністю. По-друге, PPTP є повністю застарілим і не може бути виправданий жодними міркуваннями. По-третє, вибір протоколу визначається сценарієм: мобільність вимагає MOBIKE, обхід firewall — порт 443, висока продуктивність — WireGuard або IKEv2 з AES-NI, legacy сумісність — L2TP/IPsec. По-четверте, поєднання протоколів (GRE + IPsec, L2TP + IPsec) є стандартною практикою — протоколи взаємодоповнюють один одного.

Еволюція продовжується: WireGuard спростив VPN до мінімально необхідного; QUIC/MASQUE відкриває нові можливості; постквантові алгоритми невдовзі стануть обов'язковими. Але базові принципи — інкапсуляція, автентифікація, шифрування — залишаться незмінними.
