# Слайди — Лекція 7 — Сучасний протокол WireGuard та концепція Zero Trust (ZTNA)

> Курс: Віртуальні приватні мережі (VPN)  
> Спеціальність: Інфокомунікації та мережна безпека  
> Кількість слайдів: 30

---

## Слайд 1

Курс: Віртуальні приватні мережі (VPN)

# Лекція 7

# Сучасний протокол WireGuardта концепція Zero Trust (ZTNA)

WireGuard · Noise Protocol · Cryptokey Routing · ZTNA

Спеціальність: Інфокомунікації та мережна безпека

---

## Слайд 2

## План лекції

- WireGuard: філософія мінімалізму та висока продуктивність

- Архітектура: Kernel-space vs User-space

- Криптографічний стек WireGuard

- Noise Protocol Framework

- Формат пакетів і кадрів WireGuard

- Фази та етапи встановлення з'єднання (Handshake)

- Cryptokey Routing та AllowedIPs

- Криза класичного VPN → концепція Zero Trust (ZTNA)

---

## Слайд 3

## WireGuard — що це?

WireGuard — сучасний VPN-протокол з відкритим вихідним кодом,
    розроблений Джейсоном Доненфельдом у 2016 році. Вбудований у ядро Linux з версії
    5.6 (2020), підтримується на Windows, macOS, Android та iOS.

### Ключові переваги

- ~4 000 рядків коду (проти 400 000 у OpenVPN)

- Висока швидкість передачі та мала затримка

- Проста конфігурація (як SSH-ключі)

- Стійкість до атак у публічних мережах

### Позиціонування

- Заміна IPsec для більшості сценаріїв

- Альтернатива OpenVPN з кращою продуктивністю

- Платформа для побудови mesh-мереж (Tailscale, Netmaker)

---

## Слайд 4

## Філософія мінімалізму

WireGuard свідомо відмовляється від гнучкості на користь безпеки та простоти.
    Менший код → менша поверхня атаки → легший аудит.

      | Параметр | WireGuard | OpenVPN | IPsec (strongSwan) |

      | Рядків коду (approx.) | ~4 000 | ~400 000 | ~300 000 |
      | Вибір алгоритмів | Фіксований набір | Сотні комбінацій | Гнучко + IKE |
      | Конфігурація | Мінімальна | Складна (CA/PKI) | Дуже складна |
      | Аудит безпеки | Простий | Складний | Дуже складний |
      | Вбудовано у ядро Linux | ✅ з v5.6 | ❌ (user-space) | ✅ (але складний) |

---

## Слайд 5

## Kernel-space vs User-space

### OpenVPN — User-space

          Мережевий трафік
          ↓
          Ядро ОС
          ↕ syscall (повільно)
          User-space процес
          ↕ context switch
          Ядро ОС
          ↓
          Мережевий інтерфейс

### WireGuard — Kernel-space

          Мережевий трафік
          ↓
          Ядро ОС
          ↓ без перемикань!
          wg0 (WireGuard module)
          ↓
          Мережевий інтерфейс

Kernel-space дає у 3–5 разів вищу пропускну здатність та нижчу затримку (латентність)

---

## Слайд 6

## Криптографічний стек WireGuard

WireGuard використовує фіксований, сучасний набір криптографічних примітивів.
    Ніяких «переговорів» щодо алгоритмів — одна безпечна конфігурація.

      | Функція | Алгоритм | Опис |

      | Шифрування даних | ChaCha20 | Потоковий шифр, RFC 8439 |
      | Аутентифікація (MAC) | Poly1305 | MAC-алгоритм, RFC 8439 |
      | AEAD-схема | ChaCha20-Poly1305 | Encrypt-then-MAC |
      | Обмін ключами (DH) | Curve25519 | ECDH на еліптичній кривій |
      | Хешування | BLAKE2s | Швидкий крипт. хеш (256 біт) |
      | KDF (виведення ключів) | HKDF | На основі BLAKE2s (RFC 5869) |
      | Захист від DoS | SipHash-2-4 | Хеш для cookies |

---

## Слайд 7

## ChaCha20 та Poly1305

### ChaCha20 — потоковий шифр

- Розроблений Дж. Бернштейном (2008)

- 256-бітний ключ, 96-бітний nonce

- Базується на ARX-операціях (ADD, ROTATE, XOR)

- Переваги: швидший за AES без апаратного прискорення

- Резистентний до timing-атак (рівночасові операції)

### Poly1305 — MAC

- Одноразовий аутентифікатор повідомлень

- Генерує 128-бітний тег аутентифікації

- Разом з ChaCha20 → AEAD-шифрування

- Захищає від підробки і модифікації пакетів

### AEAD = Шифрування + MAC

Authenticate-Encrypt-Authenticate:
        C = ChaCha20(K, N, P)
        T = Poly1305(K', C || AAD)

---

## Слайд 8

## Curve25519 та BLAKE2s

### Curve25519 — ECDH

- Еліптична крива Монтгомері: y² = x³ + 486662x² + x

- Розроблена Д. Бернштейном (2006)

- 256-бітні ключі ≈ безпека RSA-3072

- Обмін ключами Діффі-Хеллмана (DH)

- Базис: публічний ключ = приватний × G

- Нечутлива до атак малих підгруп

### BLAKE2s — хешування

- Оптимізований для 32-бітних платформ

- Вихід: до 256 бітів

- Швидший за SHA-256 і MD5

- Використовується у HMAC/HKDF для виведення сесійних ключів

- Захист hash-таблиць: SipHash-2-4 (проти HashDoS)

Усі ці алгоритми розраховані для 128-бітного рівня безпеки — достатньо навіть проти квантових атак типу Grover (ефективно 64 біти)

---

## Слайд 9

## Noise Protocol Framework

WireGuard будує хендшейк на базі Noise Protocol Framework (Trevor Perrin, 2016) —
    формальної специфікації для побудови безпечних протоколів обміну ключами.

### Паттерн: Noise_IKpsk2

- I — статичний ключ ініціатора (відомий заздалегідь)

- K — статичний ключ респондера відомий ініціатору

- psk2 — Pre-Shared Key (опціональний, зміцнення)

- Ефемерні ключі генеруються на кожен хендшейк

### Властивості протоколу

- Perfect Forward Secrecy (PFS) — кожна сесія має унікальні ключі

- Identity Hiding — статичний ключ ініціатора зашифрований

- Key Compromise Impersonation (KCI) resistance

- Формальна верифікація (ProVerif / Tamarin)

---

## Слайд 10

## Формат пакетів WireGuard

WireGuard визначає чотири типи повідомлень. Усі передаються через UDP.

      | Тип | Код | Призначення |

      | Handshake Initiation | 1 | Ініціатор → Респондер (початок хендшейку) |
      | Handshake Response | 2 | Респондер → Ініціатор (завершення хендшейку) |
      | Cookie Reply | 3 | Захист від DoS (cookie challenge) |
      | Transport Data | 4 | Зашифровані дані (основний трафік) |

### Encapsulation у мережевому стеку

      Ethernet
Header
      →
      IP
Header
      →
      UDP
Header
      →
      WireGuard
Header (type)
      →
      Encrypted
Payload

---

## Слайд 11

## Формат: Handshake Initiation (тип 1)

      | Поле | Розмір | Значення |

      | message_type | 1 байт | 0x01 |
      | reserved | 3 байти | 0x000000 |
      | sender_index | 4 байти | Локальний індекс сесії ініціатора |
      | unencrypted_ephemeral | 32 байти | Відкрита частина ефемерного ключа E_i |
      | encrypted_static | 48 байт | AEAD(статичний ключ S_i public + MAC) |
      | encrypted_timestamp | 28 байт | AEAD(TAI64N timestamp + MAC) |
      | mac1 | 16 байт | MAC усього повідомлення (захист) |
      | mac2 | 16 байт | Cookie MAC (якщо є DoS-виклик) |

Загальний розмір: 148 байт

---

## Слайд 12

## Формат: Handshake Response (тип 2)

      | Поле | Розмір | Значення |

      | message_type | 1 байт | 0x02 |
      | reserved | 3 байти | 0x000000 |
      | sender_index | 4 байти | Локальний індекс сесії респондера |
      | receiver_index | 4 байти | Індекс ініціатора з повідомлення 1 |
      | unencrypted_ephemeral | 32 байти | Відкрита частина ефемерного ключа E_r |
      | encrypted_nothing | 16 байт | AEAD(∅) — порожнє зашифроване поле + MAC |
      | mac1 | 16 байт | MAC усього повідомлення |
      | mac2 | 16 байт | Cookie MAC |

Загальний розмір: 92 байти

---

## Слайд 13

## Формат: Transport Data (тип 4)

      | Поле | Розмір | Значення |

      | message_type | 1 байт | 0x04 |
      | reserved | 3 байти | 0x000000 |
      | receiver_index | 4 байти | Індекс отримувача (з хендшейку) |
      | counter | 8 байт | Монотонний лічильник (захист від Replay) |
      | encrypted_encapsulated_packet | змінна | ChaCha20-Poly1305(IP-пакет) + 16-байтний MAC |

### Захист від Replay

64-бітний лічильник + sliding window (2048 пакетів)

### Padding

IP-пакет вирівнюється до кратного 16 байтам для ефективності

---

## Слайд 14

## Handshake: загальна схема

Хендшейк WireGuard складається з 2 повідомлень і встановлює двосторонні сесійні ключі.
    Весь процес займає 1 RTT (round-trip time).

      Ініціатор
(Initiator)
      Респондер
(Responder)

      1. Handshake Init →

      (тип=1, 148 байт)

      (тип=2, 92 байти)

      ← 2. Handshake Response

      3. Transport Data →

      (тип=4, змінна)

      (тип=4, змінна)

      ← 4. Transport Data

Після хендшейку: 2 незалежних ключі — один для кожного напряму (TX / RX)

---

## Слайд 15

## Фаза 1: Handshake Initiation

Ініціатор I знає публічний статичний ключ респондера S_r.

      1

        Генерація ефемерного ключа

E_i = Curve25519.generate() — унікальна пара ключів для цього хендшейку

      2

        Ланцюг хешування (chaining key)

C₀ = BLAKE2s(protocol_name)H₀ = BLAKE2s(C₀ || identifier)H₁ = BLAKE2s(H₀ || S_r_pub)

      3

        DH-операції та виведення ключів (HKDF)

(C₁, k₁) = HKDF(C₀, DH(E_i, S_r_pub))encrypted_static = ChaCha20-Poly1305(k₁, S_i_pub)(C₂, k₂) = HKDF(C₁, DH(S_i, S_r_pub))encrypted_timestamp = ChaCha20-Poly1305(k₂, timestamp)

      4

        MAC-захист і відправка

mac1 = MAC(BLAKE2s(S_r_pub), message) — аутентифікація адресата

---

## Слайд 16

## Фаза 2: Handshake Response

Респондер R перевіряє повідомлення 1 і генерує відповідь.

      1

        Верифікація Initiation

R відновлює ланцюг C та H, дешифрує encrypted_static та перевіряє mac1 й timestamp

      2

        Генерація ефемерного ключа E_r

E_r = Curve25519.generate() — новий ефемерний ключ респондера

      3

        Три додаткові DH-операції

(C₃, _) = HKDF(C₂, DH(E_r, E_i_pub))(C₄, _) = HKDF(C₃, DH(E_r, S_i_pub))(C₅, τ, k₃) = HKDF(C₄, PSK) — Pre-Shared Key (або нулі)

      4

        Виведення сесійних ключів

T_send = BLAKE2s(C₅ || "0x01") — ключ для TXT_recv = BLAKE2s(C₅ || "0x02") — ключ для RX

---

## Слайд 17

## Встановлення транспортних ключів

### Властивості сесійних ключів

- PFS — компрометація довгострокових ключів не розкриває минулі сесії

- 2 ключі на сесію — окремі для TX і RX

- ChaCha20-Poly1305 з 64-бітним лічильником

- Ключі оновлюються кожні 180 секунд (REKEY_AFTER_TIME)

- Або кожні 2⁶⁰ пакетів (REKEY_AFTER_MESSAGES)

### Таймери та стан

- REKEY_AFTER_TIME: 180 сек — оновлення ключа

- REJECT_AFTER_TIME: 180 сек — відхилення старих

- REKEY_TIMEOUT: 5 сек — retry хендшейку

- KEEPALIVE_TIMEOUT: 10 сек — keepalive

- Стан сесії: stateless — WireGuard не зберігає "активних з'єднань"

5 DH-операцій у хендшейку забезпечують взаємну аутентифікацію, PFS та приховування ідентичності

---

## Слайд 18

## Cryptokey Routing

Cryptokey Routing — унікальний механізм WireGuard, де
    маршрутизація і аутентифікація об'єднані: ключ визначає дозволені IP-адреси.

### Таблиця Cryptokey Routing

[Peer: Alice]
PublicKey = <alice_pub_key>
AllowedIPs = 10.0.0.2/32

[Peer: Bob]
PublicKey = <bob_pub_key>
AllowedIPs = 10.0.0.3/32, 192.168.1.0/24

[Peer: Gateway]
PublicKey = <gw_pub_key>
AllowedIPs = 0.0.0.0/0

### Логіка маршрутизації

        При відправці (TX):

Пошук пакету за destination IP → знаходимо peer → шифруємо його ключем

При отриманні (RX):

Дешифруємо ключем peer → перевіряємо source IP ∈ AllowedIPs → якщо ні — відкидаємо

---

## Слайд 19

## Конфігурація WireGuard

### Серверна конфігурація (wg0.conf)

[Interface]
Address    = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>
PostUp     = iptables -A FORWARD -i wg0 -j ACCEPT
PostDown   = iptables -D FORWARD -i wg0 -j ACCEPT

[Peer]
# Client Alice
PublicKey  = <alice_public_key>
AllowedIPs = 10.0.0.2/32

### Клієнтська конфігурація

        [Interface]
Address    = 10.0.0.2/32
PrivateKey = <alice_private_key>
DNS        = 1.1.1.1

[Peer]
# Server
PublicKey    = <server_public_key>
Endpoint     = vpn.example.com:51820
AllowedIPs   = 0.0.0.0/0
PersistentKeepalive = 25

  Активація: wg-quick up wg0 | Статус: wg show

---

## Слайд 20

## «Невидимість» WireGuard

WireGuard не відповідає на пакети, що не пройшли криптографічну перевірку.
    Для зовнішнього сканера порт виглядає закритим, навіть якщо WireGuard активний.

### Традиційний VPN (IPsec/OpenVPN)

- Відповідає на probe-пакети

- Порт виявляється при скануванні

- Можлива ідентифікація типу VPN

- Вразливий до DDoS на фазі хендшейку

### WireGuard — Stealth mode

- Silence is security — нульова реакція на чужі пакети

- Порт-сканери бачать порт як "filtered"

- Ускладнює fingerprinting VPN

- Cookie mechanism — захист від DoS через SipHash cookies

---

## Слайд 21

## WireGuard vs IPsec vs OpenVPN

      | Критерій | WireGuard | IPsec/IKEv2 | OpenVPN |

      | Продуктивність | ~10 Gbps | ~6 Gbps | ~1-2 Gbps |
      | Затримка | Мінімальна | Середня | Висока |
      | Handshake час | 1 RTT | 4+ RTT | 4+ RTT |
      | Криптографія | Фіксована (безпечна) | Гнучка (ризик) | Гнучка (ризик) |
      | Роуминг (NAT) | Автоматичний | MOBIKE | Обмежений |
      | Конфігурація | Проста | Складна | Середня |
      | Вбудов. у ядро | ✅ Linux 5.6+ | ✅ | ❌ |
      | UDP-порт | 51820 (змін.) | 500, 4500 | 1194 (UDP/TCP) |

---

## Слайд 22

Частина II
  🛡️

# Криза класичного VPN

# та концепція Zero Trust (ZTNA)

Never Trust, Always Verify

---

## Слайд 23

## Проблеми класичного VPN

### Модель "Замок і ров"

- Після аутентифікації — повний доступ до сегменту мережі

- Всі пристрої всередині периметру "довірені"

- Компрометація 1 акаунту → доступ до всього

### Lateral Movement

- Зловмисник переміщується горизонтально всередині мережі

- Важко виявити — трафік є "легітимним"

### Сучасні виклики

- Хмарні ресурси поза периметром

- BYOD і особисті пристрої

- Атаки на ланцюжки поставок (Supply Chain)

- Ransomware через VPN-сесії

- Витоки даних від привілейованих користувачів

97% організацій зазнали порушень безпеки, незважаючи на VPN(VPN Risk Report, Cybersecurity Insiders, 2023)

---

## Слайд 24

## Концепція Zero Trust

Zero Trust — архітектурна концепція безпеки, розроблена Джоном Кіндервагом у Forrester (2010).
    Девіз: "Never Trust, Always Verify" — ніколи не довіряй, завжди перевіряй.

### 3 Основні принципи

- Не існує периметра довіри

- Least Privilege — мінімально необхідний доступ

- Мікросегментація — ізоляція кожного ресурсу

### Що перевіряється?

- Ідентифікація користувача (MFA)

- Стан пристрою (endpoint health check)

- Контекст: час, місце, поведінка

- Безперервна верифікація — не лише при вході

---

## Слайд 25

## Zero Trust Network Access (ZTNA)

ZTNA — реалізація Zero Trust для мережевого доступу.
    Замість тунелю до мережі → прямий захищений доступ до конкретного застосунку.

### Класичний VPN

          Користувач
          ↓ аутентифікація
          VPN-тунель до мережі
          ↓
          Доступ: вся мережа ⚠️

### ZTNA

          Користувач
          ↓ ідентифікація + перевірка пристрою
          Access Broker (Connector)
          ↓ мікросегментація
          Доступ: лише App A ✅

---

## Слайд 26

## Архітектура ZTNA

### Компоненти

- Identity Provider (IdP) — Okta, Azure AD, Google Workspace

- Policy Engine — обчислює дозволи на доступ

- Policy Administrator — управляє сесіями і ключами

- Policy Enforcement Point (PEP) — застосовує рішення

- Access Broker/Connector — проксі перед застосунком

### Мікросегментація

- Кожен застосунок ізольований

- Ресурси приховані від публічного Інтернету

- Зловмисник не може знайти ціль через сканування

### Continuous Verification

- Перевірка не лише при вході, але й під час сесії

- Anomaly detection (UEBA)

- Автоматичне завершення сесії при аномалії

---

## Слайд 27

## Моделі розгортання ZTNA

### Client-initiated ZTNA

- На пристрої встановлений ZTNA-агент

- Агент аутентифікує користувача і пристрій

- Встановлює зашифрований тунель до PEP

- Приклади: Zscaler ZPA, Cloudflare Access, Tailscale

### Service-initiated ZTNA

- Без агента на пристрої

- Застосунок сам підключається до хмарного брокера

- Доступ через браузер або clientless

- Приклади: Appgate SDP, Akamai EAA

      | Критерій | Client-initiated | Service-initiated |

      | Управління пристроєм | Повний контроль | Обмежений |
      | Підтримка BYOD | Складно | Легко |
      | Перевірка health пристрою | Так | Обмежено |

---

## Слайд 28

## ZTNA vs Класичний VPN

      | Параметр | Класичний VPN | ZTNA |

      | Модель довіри | Implicit trust (після входу) | Never trust, always verify |
      | Одиниця доступу | Сегмент мережі | Конкретний застосунок |
      | Видимість ресурсів | Широка (всі ресурси мережі) | Лише дозволені застосунки |
      | Lateral movement | Легко можливий | Мікросегментація блокує |
      | Перевірка пристрою | Опціональна | Обов'язкова |
      | Продуктивність | Centralized backhauling | Direct-to-cloud |
      | Хмарна готовність | Обмежена | Cloud-native |

---

## Слайд 29

## WireGuard як основа ZTNA-рішень

Завдяки своїй простоті та продуктивності WireGuard став базовим транспортом для багатьох сучасних ZTNA-платформ.

### Приклади платформ

- Tailscale — mesh VPN на WireGuard + Magic DNS + ACL

- Netmaker — automated WireGuard management + ZTNA

- Headscale — self-hosted Tailscale control plane

- Firezone — WireGuard + ZTNA access management

### WireGuard + ZTNA = ?

- WireGuard надає транспорт (швидкий, захищений)

- Cryptokey Routing → база для мікросегментації

- Identity Provider (IdP) інтегрується поверх

- Policy Engine керує AllowedIPs динамічно

- Результат: Zero Trust Mesh Network

---

## Слайд 30

🔐

# Дякую за увагу!

Лекція 7: WireGuard та Zero Trust (ZTNA)

    WireGuard
    ChaCha20-Poly1305
    Curve25519
    BLAKE2s
    Noise Protocol
    Cryptokey Routing
    PFS
    Zero Trust
    ZTNA
    Мікросегментація
    Never Trust Always Verify

### Головні висновки

- ✅ WireGuard — мінімалістичний, швидкий, безпечний VPN-протокол (1 RTT handshake)

- ✅ Фіксований криптостек — ChaCha20/Poly1305, Curve25519, BLAKE2s, Noise

- ✅ Cryptokey Routing — маршрутизація = аутентифікація

- ⚠️ Класичний VPN — недостатній для сучасних загроз (lateral movement)

- ✅ ZTNA — "Never Trust, Always Verify", доступ лише до конкретних застосунків

Наступна лекція: Лекція 8 — Огляд хмарних VPN-рішень

&#8249;
&#8250;

---
