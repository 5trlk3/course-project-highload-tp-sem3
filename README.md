# Расчётно-пояснительная записка по курсу «Проектирование высоконагруженных систем»

## 1. Тема и целевая аудитория

**Тип сервиса**: Музыкальный стриминг (аналог Яндекс.Музыка)

**Основной функционал MVP**:

1. Регистрация и авторизация;
2. Прослушивание треков;
3. Персональные рекомендации;
4. Поиск по исполнителю и названию;
5. Пользовательские и редакторские плейлисты, персональные/жанровые ленты, похожие треки;
6. Лайки треков и подписки на артистов.

**Аудитория**: ~30,5 млн MAU; Россия — ~95%, Беларусь — ~1,5%, остальные — другие страны СНГ [^1].

## 2. Расчет нагрузки

### Продуктовые метрики

|Тип метрики | Величина|
|:--|:--|
|MAU | 30.5 млн.|
|DAU| 15-16 млн. (30.5 * 0.5) |

Для расчёта принято DAU = 15,5 млн (~50% MAU). Ориентир для Яндекс Плюса — 21 день использования в месяц [^2].

**Действия на DAU в сутки**

|Тип действия | Количество в день на пользователя|
|:--|:--|
|Воспроизведение трека| ~ 12|
|Запрос рекомендаций| ~ 3|
|Поисковый запрос| ~ 1,5|
|Действие с плейлистом или реакция (лайк/подписка)| ~ 1 суммарно|
|Авторизация / обновление сессии| ~ 3|

Принято 40 минут прослушивания в день; по РОМИР 78% слушают свыше 30 минут [^3]. Средний трек — 3,3 минуты (Spotify, 2020) [^4]; 40 / 3,3 ≈ 12 треков в сутки.

Частоты действий — допущения. Три обращения к Auth учитывают вход/refresh при JWT на 15 минут и 40 минутах прослушивания; чтение профиля входит в прикладные запросы.

### Технические метрики

**Размер хранения в разбивке по типам данных**

Каталог: 76 млн треков в 2023 году [^5] + 80 тыс./неделю [^6] × 104 недели ≈ 84,3 млн; принято 85 млн.

Для расчёта храним три качества: 64 / 192 / 320 кбит/с (HE-AAC / AAC / MP3) [^7]. При длительности 200 с: 1,6 + 4,8 + 8,0 = 14,4 МБ на трек.

|Тип данных | Количество единиц | Размер на единицу | Итого|
|:--|:--|:--|:--|
|Аудиофайлы каталога (3 версии битрейта на трек)| 85 млн треков| 14,4 МБ| ~ 1224 ТБ (1,2 ПБ)|
|Метаданные каталога и медиассылки| 85 млн треков и связанные сущности| по разделу 5| ~0,33 ТБ|
|Оперативные пользовательские данные (состояния, элементы плейлистов, история 7 суток, очереди с TTL)| по разделу 5| бюджет с запасом на вспомогательные поля| ~4,8 ТБ до индексов и репликации|
|EngineResultCache (top-100 на движок)| до 4 движков, TTL 5 минут| компактные списки| бюджет ~0,01 ТБ|
|Артефакты RecommendationModel| BPR, co-listen, CONTENT/HNSW, популярное| две версии, Ceph| бюджет до 2 ТБ|
|Изображения| ~14,2 млн| ~0,3 МБ| ~4,3 ТБ|
|**Базовый бюджет полезных данных**| | | ~1236 ТБ; без DWH, поискового индекса, временных датасетов, индексов СУБД и репликации|

|Тип метрики | Величина|
|:--|:--|
|Оперативные данные в пересчёте на MAU|~0,16 МБ, усреднённый бюджет|
|Хранилище на 1 пользователя, оффлайн-кэш|0.25 Гб|

Офлайн-кэш 0,25 ГБ хранится на устройстве и в серверный бюджет не входит.

**RPS в разбивке по типам запросов**

Средний RPS = DAU × действий в день / 86 400. Для трека 200 с и heartbeat раз в 30 с принято 6 heartbeat + 1 finish сверх start: 12 × 7 = 84 запроса на DAU в сутки.

|Тип запроса | Средний RPS | Пиковый RPS|
|:--|:--|:--|
|Воспроизведение трека| ~ 2153| ~ 5382|
|Запрос рекомендаций| ~ 538| ~ 1345|
|Поиск| ~ 269| ~ 673|
|Операции с плейлистом и реакции суммарно| ~ 179| ~ 448|
|Авторизация| ~ 538| ~ 1345|
|Playback heartbeat + finish| ~15 069| ~37 674|
|**Итого API**| ~18 747| ~46 868|

Итоги вычислены до округления строк.

**Сетевой трафик**

Доли качеств приняты 20/60/20%: 0,2\*64 + 0,6\*192 + 0,2\*320 = 192 кбит/с.

Время прослушивания: 15,5 млн × 40 мин = 620 млн человеко-минут = 37,2 млрд человеко-секунд/сутки. Средний поток: 37,2 × 10^9 × 192 000 / 86 400 ≈ 82,7 Гбит/с; объём — ~893 ТБ/сутки.

Коэффициент пик/среднее принят 2,5. Пиковый аудиотрафик: 82,7 × 2,5 ≈ 207 Гбит/с.

Payload: start — 2 КБ, рекомендации — 10 КБ, поиск — 5 КБ, прочие — 1 КБ. API без heartbeat/finish: ~235 Мбит/с и 1,015 ТБ/сутки; heartbeat/finish: ~301 Мбит/с и 1,302 ТБ/сутки. Заголовки и повторы учтены не полностью.

Изображения: 5 загрузок по 50 КБ на DAU — ~3,88 ТБ/сутки и ~0,9 Гбит/с в пике. Медиатрафик не входит в API RPS.

|Тип трафика | Пиковое потребление, Гбит/с | Суммарный суточный объем, ТБ/сутки|
|:--|:--|:--|
|Аудиопоток (стриминг)| ~ 207| ~ 893|
|API, включая playback-телеметрию| ~0,54| ~2,32|
|Изображения через CDN| ~0,9| ~3,88|
|**Итого**| ~208,4| ~899,2|

### Прирост метрик

|Метрика | Сейчас | Прирост за год|
|:--|:--|:--|
|MAU| 30,5 млн| +32% (~ +9,8 млн; ~ 27 тыс./сутки)|
|DAU| 15,5 млн| +32% (~ +5,0 млн; ~ 14 тыс./сутки)|
|Каталог| 85 млн треков| +4,9% (~ +4,16 млн; ~ 11,4 тыс./сутки)|
|Аудиофайлы каталога| ~ 1224 ТБ| ~ +60 ТБ (~ 0,16 ТБ/сутки)|
|Метаданные каталога| ~0,33 ТБ| ~+0,016 ТБ при росте каталога на 4,9%|
|Пользовательские данные| ~ 4,8 ТБ| +32% (~ +1,54 ТБ; ~ 4,2 ГБ/сутки)|
|Кэш движков / артефакты моделей| до 0,01 / 2 ТБ| кэш растёт с RPS, артефакты — с моделью/каталогом|
|**Базовое полезное хранилище**| ~1236 ТБ| ~+62 ТБ/год без DWH/индексов/реплик; рост изображений и соседей условно пропорционален каталогу|
|Пиковый сетевой трафик (аудио)| ~ 207 Гбит/с| +32% (~ +66 Гбит/с)|
|Суточный объем трафика| ~899,2 ТБ/сутки| +32% (~+288 ТБ/сутки)|
|Средний API RPS| ~18 747| +32% (~+5999)|
|Пиковый API RPS| ~46 868| +32% (~+14 998)|

Прогноз роста аудитории — 32%/год [^1]. Накопление playback-DWH вырастет со 108,6 до ~143,4 ТБ/год без индексов и служебных данных.

## 3. Глобальная балансировка нагрузки

**Функциональное разбиение по доменам**

|Домен | Описание | Требования|
|:--|:--|:--|
|Auth| учётные данные, access/refresh, AuthSession| ротация и отзыв сессий|
|Users| профиль и план пользователя| независим от сценария входа|
|Catalog & Search| треки, артисты, медиа, поиск| CatalogIndexWorker обновляет индекс|
|Recommendations| плейлисты, конфиги движков, merger/ranker| генерация по запросу, короткий engine cache|
|Playback| аудиодоступ, история, context/queue| стабильные страницы очереди|
|Reactions| личные лайки и подписки| согласованное состояние + CDC|
|Event platform| приём событий, проекции счётчиков| Kafka, projector/aggregator workers|
|Analytics| DWH, исторические снимки, датасеты| асинхронная загрузка и export Cron|

**Обоснование расположения ДЦ**

~70% населения РФ живёт в европейской части [^9]. Core ДЦ размещаем в ЦФО рядом с MSK-IX [^8]; кластер основных ДЦ Яндекса также находится в центральной России [^10].

Три core ДЦ работают active-active-active. CDN — Москва, Петербург, Екатеринбург, Новосибирск; пиринг — Минск и Алматы.

**Расчет распределения запросов по ДЦ**

На core ДЦ штатно приходится ~6249/15 623 среднего/пикового RPS, при отказе одного — ~9374/23 434.

При допущении 90% попаданий по байтам CDN отдаёт ~186 Гбит/с, Ceph — ~21 Гбит/с: по ~7 Гбит/с на core ДЦ штатно и ~10,5 при отказе одного. Для изображений дополнительно резервируется до 0,9 Гбит/с origin-трафика.

**Схема DNS-балансировки (GeoDNS)**

```
Клиент -> локальный DNS-резолвер -> авторитативный GeoDNS
       -> определяет регион резолвера и живой (health-check) ДЦ
       -> возвращает IP ближайшего/доступного core ДЦ (ДЦ-1, ДЦ-2 или ДЦ-3)
Клиент -> подключается напрямую по полученному IP
```

Переключение GeoDNS ограничено TTL резолвера и клиента [^11].

**Схема Anycast-балансировки**

CDN анонсирует общий IP-префикс из точек присутствия; точку обслуживания определяет BGP-маршрутизация [^11]. При отказе анонс отзывается. Сходимость BGP занимает время; соединения могут потребовать переподключения.

**Механизм регулировки трафика между ДЦ**

- GeoDNS исключает недоступные ДЦ по health-check; TTL — 30–60 с.
- При плановых работах трафик переводится постепенно через DNS/BGP.

## 4. Локальная балансировка нагрузки

API — stateless pod в Kubernetes. L7 — **NGINX Ingress Controller** (nginx/kubernetes-ingress, OSS) [^19]; L4-правила Service — **kube-proxy**, режим nftables [^20].

**Уровни и резервирование**

```
Клиент -> L4-правила входного Service (kube-proxy/nftables) -> L7 (NGINX Ingress Controller, TLS termination, роутинг)
       -> App-под домена -> межсервисные вызовы (sidecar proxy) -> нужная СУБД (раздел 6)
```

Ingress направляет запросы к pod-адресам сервиса; kube-proxy настраивает L4-правила.

|Уровень | Формула | Что переживает|
|:--|:--|:--|
|L4 / L7 / App-под / read-реплика (внутри ДЦ)| N+1| отказ одного узла|
|3 core ДЦ (раздел 3, между ДЦ)| N+1 на уровне ДЦ| отказ одного ДЦ целиком|

N+1 — рабочая мощность плюс запасной элемент. Каждый из трёх ДЦ рассчитан на 50% пика: суммарно 150%, любые два выдерживают 100%. Для двух ДЦ потребовалось бы по 100%, суммарно 200%.

**L3/L4 перед CDN: LVS/IPVS**

|Способ | Ответ сервера | Нагрузка на LB|
|:--|:--|:--|
|NAT| идёт обратно через LB| высокая|
|IP Tunneling | идёт клиенту напрямую| только входящий трафик, но есть накладные расходы на инкапсуляцию/декапсуляцию на каждый пакет|
|Direct Routing (DR, меняется MAC, не IP)| идёт клиенту напрямую| LB обрабатывает входящие пакеты, объёмный ответ обходит его|

Выбран DR: ответ с медиатрафиком ~207,9 Гбит/с обходит LB. LVS и CDN-узлы — в общей L2-сети, вне Kubernetes; keepalived переносит VIP по VRRP. Входящие PPS/CPS проверяются отдельно.

**L7**: least_conn для поиска с разным временем ответа; преимущество перед round-robin проверяется нагрузкой.

**Межсервисные вызовы**: Envoy sidecar получает готовые pod через headless Service / CoreDNS и балансирует вызовы напрямую.

**Схема отказоустойчивости**

|Что отказало | Кто обнаруживает | Что происходит|
|:--|:--|:--|
|L4/L7/app-узел| health-check| узел исключается, N+1 покрывает|
|Read-реплика БД| health-check| чтение уходит на оставшиеся (N+1)|
|Весь core ДЦ| внешние health-check, обновление GeoDNS (раздел 3)| оставшиеся 2 из 3 ДЦ принимают по 50% пика (N+1); переключение зависит от DNS-кэшей и доступности реплик данных|
|CDN-сервер / вся edge-площадка| локальные проверки / отзыв BGP-анонса| сервер исключается локально; при потере площадки маршруты сходятся к другой, клиенту может потребоваться повторное подключение|

**Расчет количества балансировщиков**

*API.* Принят ориентир ~10 000 новых HTTPS-соединений/с на узел [^16][^17]. При одном соединении на запрос: ceil(23 434 / 10 000) = 3 рабочих + 1 резервный Ingress на ДЦ, всего 12. Keep-alive/resumption снижают CPS [^18].

*CDN.* При 10 Гбит/с на узел: ceil((207 + 0,9) / 10) = 21 рабочий сервер. По одному резервному на четыре площадки: 21 + 4 = 25 (10/7/4/4). Cache hit уменьшает origin-трафик, но не исходящий поток CDN.

## 5. Логическая схема БД

FK на схеме — логические ссылки между доменами.

```mermaid
erDiagram
    USER ||--o{ AUTH_IDENTITY : identifies
    USER ||--o{ AUTH_SESSION : authenticates
    AUTH_IDENTITY ||--o{ AUTH_SESSION : opens
    AUTH_SESSION ||--o{ PLAYBACK_SESSION : authorizes
    USER ||--o| ARTIST_PROFILE : has
    ARTIST_PROFILE ||--o{ TRACK : publishes
    ARTIST_PROFILE ||--o{ ALBUM : releases
    ALBUM o|--o{ TRACK : contains
    TRACK ||--o{ AUDIO_VARIANT : offers
    MEDIA_ASSET ||--o| AUDIO_VARIANT : stores_audio
    MEDIA_ASSET o|--o{ USER : avatar
    MEDIA_ASSET o|--o{ ARTIST_PROFILE : photo
    MEDIA_ASSET o|--o{ ALBUM : cover
    MEDIA_ASSET o|--o{ TRACK : cover
    TRACK ||--o| TRACK_SEARCH_DOCUMENT : indexed_as
    USER ||--o{ USER_TRACK_LIKE : sets
    TRACK ||--o{ USER_TRACK_LIKE : receives
    USER ||--o{ USER_ARTIST_FOLLOW : follows
    ARTIST_PROFILE ||--o{ USER_ARTIST_FOLLOW : followed_by
    TRACK ||--|| TRACK_COUNTERS : has
    ARTIST_PROFILE ||--|| ARTIST_COUNTERS : has
    TRACK_COUNTERS o|--o{ COUNTER_SNAPSHOT : track_history
    ARTIST_COUNTERS o|--o{ COUNTER_SNAPSHOT : artist_history
    USER ||--o{ PLAYBACK_SESSION : starts
    TRACK ||--o{ PLAYBACK_SESSION : played_in
    USER ||--o{ USER_ACTIVITY_HISTORY : reads
    PLAYBACK_SESSION o|--o{ USER_ACTIVITY_HISTORY : playback_actions
    USER o|--o{ DOMAIN_EVENT : actor
    PLAYBACK_SESSION o|--o{ DOMAIN_EVENT : playback_facts
    DOMAIN_EVENT ||--o| USER_ACTIVITY_HISTORY : projected_to
    USER ||--o{ PLAYLIST : authors
    MEDIA_ASSET o|--o{ PLAYLIST : cover
    PLAYLIST ||--o{ PLAYLIST_ITEM : embeds
    TRACK ||--o{ PLAYLIST_ITEM : appears_in
    USER ||--o{ PLAYBACK_CONTEXT : opens
    PLAYLIST o|--o{ PLAYBACK_CONTEXT : static_source
    FEED_DEFINITION o|--o{ PLAYBACK_CONTEXT : feed_source
    TRACK o|--o{ PLAYBACK_CONTEXT : seed
    PLAYBACK_CONTEXT ||--o{ PLAYBACK_QUEUE_ITEM : buffers
    TRACK ||--o{ PLAYBACK_QUEUE_ITEM : queued
    PLAYBACK_CONTEXT ||--o{ PLAYBACK_SESSION : played_as
    TRACK ||--o{ TRACK_CONTENT_EMBEDDING : content_vector
    RECOMMENDATION_MODEL ||--o{ TRACK_CONTENT_EMBEDDING : content_version
    RECOMMENDATION_MODEL o|--o{ ENGINE_RESULT_CACHE : caches_version
    USER o|--o{ ENGINE_RESULT_CACHE : personalized

    USER {
        uuid user_id PK
        string display_name
        string subscription_status
        uuid avatar_media_id FK
    }
    AUTH_IDENTITY {
        string identity_id PK "local:email или provider:subject"
        uuid user_id FK
        string password_hash "для local"
        string status "PENDING или ACTIVE"
    }
    AUTH_SESSION {
        uuid auth_session_id PK
        uuid user_id FK
        string identity_id FK
        string refresh_token_hash
        bigint version
        string device_id
        datetime expires_at
        datetime revoked_at
    }
    ARTIST_PROFILE {
        uuid artist_user_id PK, FK
        string stage_name
        string description
        uuid photo_media_id FK
    }
    ALBUM {
        uuid album_id PK
        uuid artist_user_id FK
        string title
        uuid cover_media_id FK
    }
    TRACK {
        uuid track_id PK
        uuid artist_user_id FK
        uuid album_id FK
        string title
        string artist_name
        string album_name
        string genre
        int duration_sec
        uuid cover_media_id FK
        string availability_status
        bigint version
    }
    MEDIA_ASSET {
        uuid media_id PK
        string media_type "AUDIO или IMAGE"
        string object_key UK
        string mime_type
        bigint size_bytes
        string checksum
        string status
    }
    AUDIO_VARIANT {
        uuid track_id PK, FK
        int bitrate PK
        uuid media_id FK
        string codec
        string status
    }
    TRACK_SEARCH_DOCUMENT {
        uuid track_id PK, FK
        string title
        string artist_name
        string album_name
        string genre
        string availability_status
        float popularity_score
        bigint source_version
        bigint signals_version
    }
    USER_TRACK_LIKE {
        uuid user_id PK, FK
        uuid track_id PK, FK
        boolean is_liked
        bigint version
        uuid request_id
        datetime updated_at
    }
    USER_ARTIST_FOLLOW {
        uuid user_id PK, FK
        uuid artist_user_id PK, FK
        boolean is_following
        bigint version
        uuid request_id
        datetime updated_at
    }
    TRACK_COUNTERS {
        uuid track_id PK, FK
        bigint like_count
        bigint qualified_play_count
        bigint projection_version
    }
    ARTIST_COUNTERS {
        uuid artist_user_id PK, FK
        bigint follower_count
        bigint projection_version
    }
    PLAYBACK_SESSION {
        uuid session_id PK
        uuid auth_session_id FK
        uuid user_id FK
        uuid track_id FK
        uuid context_id FK
        string status
        bigint last_sequence
        int position_ms
        int listened_ms
        boolean play_counted
        datetime last_heartbeat_at
    }
    USER_ACTIVITY_HISTORY {
        uuid user_id PK, FK
        string event_type PK
        date bucket PK
        datetime occurred_at PK
        uuid event_id PK, FK
        uuid session_id FK
        uuid track_id FK
        uuid artist_user_id FK
        json payload
    }
    DOMAIN_EVENT {
        uuid event_id PK
        string event_type
        string aggregate_key
        uuid actor_user_id FK "nullable для системных событий"
        uuid session_id FK
        uuid track_id FK
        uuid artist_user_id FK
        bigint sequence_or_state_version
        datetime occurred_at
        datetime received_at
        json payload
    }
    COUNTER_SNAPSHOT {
        string entity_type PK "TRACK или ARTIST"
        uuid entity_id PK
        date snapshot_date PK
        json counters
        string watermark
        datetime computed_at
    }
    PLAYLIST {
        uuid playlist_id PK
        uuid owner_user_id FK
        string title
        uuid cover_media_id FK
        string visibility
        bigint revision
    }
    PLAYLIST_ITEM {
        uuid playlist_id PK, FK
        uuid item_id PK
        string position_key
        uuid track_id FK
        datetime added_at
    }
    FEED_DEFINITION {
        uuid feed_id PK
        string title
        string source_type
        json engines "движки, лимиты, веса"
        json ranker "RRF или NONE"
        json filters
        bigint version
    }
    PLAYBACK_CONTEXT {
        uuid context_id PK
        uuid user_id FK
        string context_type "TRACK/STATIC_PLAYLIST/FEED/SIMILAR"
        uuid playlist_id FK
        uuid feed_id FK
        uuid seed_track_id FK
        bigint source_version
        string source_cursor
        string cursor
        list seen_track_ids "до 100, динамические ленты"
        datetime expires_at
    }
    PLAYBACK_QUEUE_ITEM {
        uuid context_id PK, FK
        bigint ordinal PK
        uuid track_id FK
        string reason
    }
    RECOMMENDATION_MODEL {
        uuid model_id PK
        string engine_type "COLLAB/CONTENT/POPULAR"
        string version
        string weights_key
        string auxiliary_key "co-listen или HNSW"
        string status
        datetime created_at
    }
    TRACK_CONTENT_EMBEDDING {
        uuid track_id PK, FK
        uuid model_id PK, FK
        vector embedding "только контентные признаки"
    }
    ENGINE_RESULT_CACHE {
        string cache_key PK
        string engine_type
        uuid user_id FK "nullable"
        uuid model_id FK "nullable"
        bigint config_version
        list ranked_candidates "до 100, TTL 5 минут"
        datetime expires_at
    }
```

Access JWT — 15 минут, refresh-сессия — до 30 суток. Ротация refresh — по version, logout отзывает сессию. Auth API проверяет срок/отзыв независимо от TTL-очистки; выданный JWT действует до истечения [^42].

Playlist.items — до 10 000 элементов; массив и revision обновляются атомарно, повторы треков разрешены. Очередь до 200 треков принадлежит Playback.

### 5.1. События и DWH

|Тип события | Основные данные | Назначение|
|:--|:--|:--|
|PLAYBACK_STARTED| session_id, auth_session_id, track_id, context_id| начало воспроизведения|
|PLAYBACK_HEARTBEAT / FINISHED| session_id, sequence, listened_ms, position_ms| прогресс / завершение|
|TRACK_PLAY_COUNTED| event_id, session_id, track_id| один факт прослушивания после min(30 сек, длительность), без перемотки|
|TRACK_LIKE_CHANGED / ARTIST_FOLLOW_CHANGED| user_id, объект, состояние, version| реакции|
|COUNTER_UPDATED / COUNTER_SNAPSHOT| объект, абсолютные значения, version/watermark| онлайн-проекции и исторические снимки|
|CATALOG_CHANGED / MODEL_READY| объект/модель, version| поиск / публикация артефактов|

MODEL_READY не включён в объём playback-архива.

### 5.2. Объёмы и нагрузка

Объёмы без индексов и реплик; реакция / элемент плейлиста — 80 байт.

|Сущности | Основание | Объём|
|:--|:--|:--|
|User / ArtistProfile / Album| 30,5 млн / 5,7 млн / 8,5 млн| ~30 / 2,3 / 3,4 ГБ|
|AuthIdentity / AuthSession| 30,5 млн × 512 байт / до 2 сессий на MAU × 256 байт| ~15,6 / 15,6 ГБ|
|Track| 85 млн × 2 КБ| ~0,17 ТБ|
|MediaAsset / AudioVariant| 269,2 млн × 256 байт / 255 млн × 128 байт| ~68,9 / 32,6 ГБ|
|Аудио / изображения| три аудиокачества / 14,2 млн изображений| ~1224 / 4,3 ТБ|
|TrackSearchDocument| до 85 млн документов| проектный индекс до 1 ТБ|
|UserTrackLike / UserArtistFollow| 250 лайков / 10 подписок на MAU| ~0,61 ТБ / 24,4 ГБ|
|Playlist с items| 10 плейлистов на MAU, по 50 элементов| ~61 ГБ метаданных + 1,22 ТБ элементов|
|UserActivityHistory| playback × 7 суток × 200 байт| ~2,08 ТБ плюс реакции|
|PlaybackContext / QueueItem| до 2 на DAU, 200 × 32 байта + до 100 seen ID, TTL 24 часа| до ~0,25 ТБ плюс метаданные|
|EngineResultCache| 1345 RPS × 300 сек × до 4 движков × 100 × 20 байт| до ~3,23 ГБ сырого payload; бюджет 10 ГБ|
|RecommendationModel и артефакты| BPR-факторы, co-listen top-50, content-векторы/HNSW, популярное; две версии| бюджет до 2 ТБ в Ceph|
|TrackContentEmbedding| 85 млн × 128 × 4 байта; внутри артефактов CONTENT| ~43,5 ГБ на версию; запас до 2 КБ/трек|
|DomainEvent, DWH| 186 млн воспроизведений/сутки × 8 событий × 200 байт| ~108,6 ТБ/год плюс производные события и реакции|
|TrackCounters / ArtistCounters / CounterSnapshot| агрегаты / ежедневные снимки| зависят от активности и срока хранения|

## 6. Физическая схема БД

MongoDB выбрана для доступа по ключу/документу; междоменные JOIN/FK не нужны. FeedDefinition и Playlist.items — документы с атомарной проверкой revision [^43]. Домены имеют отдельные БД/ACL на общем кластере [^44].

|Данные / владелец | Размещение и шардирование | Индексы / согласованность|
|:--|:--|:--|
|AuthIdentity / Auth| MongoDB auth: hashed _id=нормализованный provider:subject или local:email| уникальность _id; user_id; majority+journal|
|AuthSession / Auth| MongoDB auth: hashed _id=auth_session_id| user_id; TTL expires_at; CAS version для refresh; API проверяет expires_at/revoked_at|
|User / Users| MongoDB users: hashed _id=user_id| чтение по ID; majority+journal|
|ArtistProfile / Album / Track / MediaAsset / Catalog| MongoDB catalog: hashed _id сущности; AudioVariant вложены в Track.variants| artist_user_id, album_id; уникальный bitrate внутри трека проверяется атомарным обновлением|
|Playlist / FeedDefinition / RecommendationModel / Recommendations| MongoDB recommendations: hashed _id=playlist_id/feed_id/model_id| owner_user_id; items и revision в одном документе; артефакты по object_key в собственном Ceph-бакете|
|TrackSearchDocument / Catalog| OpenSearch: 12 primary shards, routing=hash(track_id), 2 replica каждого| zone awareness по трём ДЦ; API только читает, CatalogIndexWorker пишет; версии metadata/signals отдельно|
|UserTrackLike / UserArtistFollow / Reactions| Cassandra: token ring, partition=user_id; clustering=track_id/artist_user_id| RF=3 по ДЦ, QUORUM; изменения через LWT/SERIAL; version/request_id [^28]|
|PlaybackSession / Playback| Cassandra: partition=session_id| RF=3, QUORUM; sequence; TTL 7 суток|
|UserActivityHistory / Playback| Cassandra: partition=(user_id,event_type,day), clustering=(occurred_at,event_id)| дневные партиции, TTL 7 суток; запись только projector worker|
|PlaybackContext / QueueItem / Playback| Redis Cluster: hash tag {context_id}, ordinal| страница + cursor атомарны в одном slot; TTL 24 часа; утрата → новый контекст|
|EngineResultCache / Recommendations| Redis Cluster: hash(engine,config/model version,user/seed,filters,signals_version,source_revision,source_cursor,exclude_hash)| до 100 кандидатов/ключ; TTL 5 минут, cache-aside; это не база готовых рекомендаций|
|TrackCounters / ArtistCounters / Event platform| Redis Cluster: ключ entity_type:entity_id:bucket; hot-объекты — до 16 buckets в разных slots| CounterAggregatorWorker читает Kafka; материализованный итог обновляется раз в секунду; read API не считает события|
|DomainEvent / Event platform + DWH| Kafka: ключ агрегата; playback.raw — 192 partitions, reactions/counted — по 96; ClickHouse: hash(aggregate_key) % 3, PARTITION BY toYYYYMM(occurred_at)| RF=3, minISR=2, acks=all; Kafka 7 суток; DWH 1 год, ORDER BY(event_type,toDate(occurred_at),aggregate_key,event_id)|
|CounterSnapshot / Analytics| ClickHouse: hash(entity_id) % 3, PARTITION BY месяц snapshot_date| ORDER BY(type,id,date); watermark; 3 реплики шарда|
|TrackContentEmbedding / Recommendations| Parquet по hash(track_id) % 64 и model_version в Ceph; HNSW в RAM движка| отдельная CONTENT-модель, BPR-факторы в другом артефакте|
|Аудио / изображения / Catalog| Ceph: object_key → CRUSH, 12 OSD-хостов| size=3, min_size=2; копии в разных ДЦ; CDN LFU|

MongoDB: **2 шарда × 3 реплики** по ДЦ, 3 config server всего, 2 mongos/ДЦ; majority и выборы primary [^30]. Balancer распределяет chunks по hashed-ключам [^43]. Запросы без shard key, включая выборки по артисту/владельцу, обращаются к обоим шардам.

Cassandra: 9 узлов, token-aware routing, NetworkTopologyStrategy — по копии в каждом ДЦ, RF=3. Redis: 6 primary + 6 replica с anti-affinity по ДЦ и ACL доменов. ClickHouse: 3 шарда × 3 реплики.

CDC сохраняет checkpoint; event_id = hash(aggregate_key, version). Playback API подтверждает STARTED в Kafka перед выдачей аудиоссылки; EventIngest принимает heartbeat/finish [^27].

## 7. Алгоритмы

### 7.1. Генерация рекомендаций

**Офлайн:**

1. FeatureExport Cron готовит из DWH засчитанные прослушивания и лайки; накопленные значения listened_ms не суммируются повторно.
2. CollabTrain обучает **BPR-MF, 128 FP32** на GPU: SGD повышает u·v_positive − u·v_negative [^45]. Сохраняются факторы пользователей/треков COLLAB. CoListen Cron строит top-50 соседей по совместным прослушиваниям последних 20 треков пользователя; версия готова после обоих артефактов.
3. ContentIndex Cron: **TF-IDF метаданных/жанров → TruncatedSVD(128) → нормализация → HNSW cosine**. SVD обучается на выборке, каталог преобразуется батчами. Результат — TrackContentEmbedding [^46].
4. Popularity Cron считает top за 24 часа. Артефакты пишутся в Ceph; ModelPublishWorker по MODEL_READY обновляет движки по ДЦ с fallback.

**Онлайн-движки:** до 100 кандидатов от каждого.

|Движок | Генерация кандидатов|
|:--|:--|
|BPR| до 1000 co-listen соседей последних 20 положительных треков + до 1000 популярных; вычислить r_cf(t)=u·v_t только на этом пуле, выбрать top-100; без ANN и полного сканирования каталога|
|CoListen| top соседей seed-трека по совместным прослушиваниям|
|ContentANN| HNSW по контентному вектору seed / до 5 недавних положительных треков|
|Popular| актуальный жанровый/общий top|
|FollowedArtists| новые треки подписанных артистов через Reactions/Catalog API|
|StaticPlaylist| следующая часть Playlist.items по порядку и revision|

FeedDefinition задаёт движки, лимиты/веса, фильтры и ranker:

|Запрос | Движки / веса | Итоговое ранжирование|
|:--|:--|:--|
|STATIC_PLAYLIST| StaticPlaylist| NONE, сохраняется порядок|
|PERSONALIZED| BPR:3, ContentANN:2, FollowedArtists:1, Popular:1| weighted RRF|
|GENRE| BPR:2, ContentANN:1, Popular:2; genre обязателен| weighted RRF|
|TRENDING| Popular| порядок популярности|
|SIMILAR| ContentANN:3, CoListen:2, Popular:1| weighted RRF|
|TRACK| выбранный трек; продолжение отдельным SIMILAR-запросом| NONE|

**Онлайн-конвейер:**

1. Выбрать FeedDefinition, получить сигналы через History/Reactions API. Результаты движков — в EngineResultCache с TTL 5 минут; промах запускает расчёт.
2. Merger удаляет дубли по track_id; Catalog API проверяет доступность/жанр. Исключаются выданные треки и очередь. STATIC идёт по item_id/source_cursor, сохраняя намеренные повторы.
3. Финальный ranker: Score(t)=Σ w_engine/(60+rank_engine(t)), отсутствующий трек даёт 0 [^47]. r_cf используется только внутри BPR при отборе кандидатов. Для STATIC/TRACK RRF отключён.
4. Выдать до 20 треков; в динамической ленте — не более двух подряд одного артиста. При нехватке — Popular с теми же фильтрами, затем конец.
5. Playback API атомарно дополняет свою очередь до 200 элементов вместе с cursor; повтор cursor возвращает ту же страницу. Смена revision статичного плейлиста обновляет контекст; динамическая следующая порция может использовать свежую модель.

Личные сигналы обновляются онлайн, без ожидания обучения BPR.

### 7.2. Вытеснение кэша CDN

**Алгоритм**: LFU по частоте обращений за последние 24 часа.

1. Ключ объекта — track_id + битрейт; при запросе обновляется частота.
2. При промахе файл получается из Ceph, отдаётся клиенту и кэшируется.
3. При нехватке места удаляются наименее частые объекты; при равенстве — давно не использованные.
4. Устаревшие обращения исключаются из статистики окна.

**Структуры**: хэш-таблица объектов, часовые корзины счётчиков, min-heap для вытеснения. Накладные расходы — память статистики и обновление приоритетов.

**Альтернативы**: LRU — давность, FIFO — порядок добавления. LFU выбран для удержания популярных треков; hit ratio проверяется на трассе запросов.

## 8. Технологии

|Категория | Технология | Назначение | Альтернативы | Обоснование|
|:--|:--|:--|:--|:--|
|Оркестрация и деплой|Kubernetes|stateless app-слой|VM / Docker Compose|размещение, перезапуск и масштабирование pod; ingress устанавливается отдельно|
|Оркестрация и деплой|GitLab CI + ArgoCD|CI/CD в трёх ДЦ|ручной kubectl apply|сборка/проверки и воспроизводимая доставка конфигурации из Git|
|Виртуализация|KVM / libvirt|два MongoDB-шарда либо Kafka/OpenSearch на общем хосте|отдельные bare-metal узлы|выделенные vCPU/RAM и диски; реплики одного шарда на разных хостах [^39]|
|Обучение|PyTorch / TorchRec / CUDA|BPR-MF на GPU|CPU-обучение|128-компонентные факторы COLLAB; SGD и шардирование [^37][^45]|
|Балансировка и сеть|NGINX Ingress Controller (nginx/kubernetes-ingress, OSS)|входящий HTTP(S)|Traefik / HAProxy|TLS, маршрутизация, least_conn; выбран поддерживаемый проект NGINX [^19]|
|Балансировка и сеть|kube-proxy, nftables|Kubernetes Service|iptables / eBPF-CNI|L4-правила на совместимых Linux-узлах; IPVS-режим kube-proxy устарел [^20]|
|Балансировка и сеть|LVS/IPVS DR + keepalived|балансировка CDN-площадки|NAT / Tunneling|ответ обходит LB; общая L2-сеть, VRRP-переключение VIP|
|Балансировка и сеть|GeoDNS|выбор core ДЦ|Anycast|простота для трёх площадок; переключение ограничено DNS TTL|
|Балансировка и сеть|Anycast/BGP|выбор CDN-площадки|GeoDNS|смена маршрута без DNS TTL; остаётся задержка сходимости BGP|
|Балансировка и сеть|Sidecar proxy (Envoy)|межсервисные вызовы|клиентская балансировка|общие политики тайм-аутов/балансировки; требует discovery адресов экземпляров|
|Балансировка и сеть|CoreDNS|DNS-имена сервисов|Consul|использует сведения Kubernetes; загрузку pod не измеряет|
|Хранилища|MongoDB|Auth, Users, Catalog, конфиги/плейлисты Recommendations|PostgreSQL|2 шарда; отдельные БД доменов; атомарные документы Playlist.items/revision [^43]|
|Поиск|OpenSearch|TrackSearchDocument|индекс MongoDB / Elasticsearch|текстовый индекс, жанровые фильтры; перестраивается из каталога [^29]|
|Хранилища|Cassandra + CDC-потребитель|лайки, подписки, сессии, история|PostgreSQL / распределённый KV-store|доступ по пользовательским ключам; LWT/QUORUM для реакций, TTL истории [^28]|
|Хранилища|Redis Cluster|кэш движков, Playback context/queue, проекции счётчиков|MongoDB|разные ACL/префиксы; TTL, hash slots, атомарные страницы [^26]|
|Хранилища|Kafka|события и changelog|прямая запись в проекции|входные события хранятся 7 суток; changelog состояния — compact; порядок по агрегату и replay [^27]|
|Обработка событий|Kafka Streams|дедупликация и проекции|Flink / собственные потребители|состояние с changelog; внешние проекции записываются идемпотентно [^27]|
|Хранилища|ClickHouse|DomainEvent, CounterSnapshot, датасеты|batch-движок|колоночные чтения, 3 шарда × 3 реплики|
|Хранилища|ClickHouse Keeper|координация реплик|ZooKeeper|компонент экосистемы ClickHouse; требуется отказоустойчивый кворум [^21]|
|Хранилища|Ceph RGW|аудио и изображения|managed S3|контроль размещения/репликации, S3 API; самостоятельная эксплуатация|
|Кэширование|собственный HTTP-кэш на NVMe, LFU с окном|CDN|Varnish LRU / NGINX proxy_cache|частота удерживает хиты; LFU реализуется отдельно, штатный Varnish использует LRU [^22]|
|Рекомендации|TF-IDF / TruncatedSVD / HNSW|отдельный ContentANN engine|Annoy / точный cosine|контентные признаки независимо от BPR; онлайн top-100 [^23][^46]|
|Доступ|OAuth 2.0 / OIDC, JWT + refresh-сессии|Auth API; профиль отдельно в Users|серверные access-сессии|JWT 15 мин, AuthSession до 30 дней; отзыв refresh и проверка срока [^24][^42]|
|Эксплуатация|Prometheus + Grafana + Alertmanager|метрики и оповещения|Zabbix|сбор, визуализация и доставка тревог; failover выполняют сами компоненты [^25]|
|Эксплуатация|ELK/EFK или Loki|централизованные логи|Graylog / OpenSearch|единый поиск ошибок по сервисам и ДЦ|
|Тестирование|Yandex.Tank / k6 / Gatling|нагрузочные испытания|собственный генератор|воспроизводимые RPS/CPS, p95/p99 и доля ошибок|
|Тестирование|Chaos Mesh|эксперименты со сбоями|собственные сценарии|проверка pod/узлов/сети; отказ ДЦ требует отдельных сценариев|

## 9. Обеспечение надёжности

Резервирование и переключение — разделы 3, 4, 6 и 11.

### 9.1. Асинхронные потоки и повторы

|Паттерн / поток | Обработка сбоя|
|:--|:--|
|CDC: реакции, каталог, конфиги| checkpoint сохраняется после ack Kafka. При выходе за окно CDC/oplog — переснимок и сверка; потерянная история изменений из текущего состояния не восстанавливается|
|Backpressure: события плеера| ограниченные буферы и 429/503 при перегрузке; клиент повторяет прежний event_id с backoff/jitter|
|Idempotent Consumer: история, счётчики| exactly_once_v2 фиксирует offset, состояние и выходные события внутри Kafka. Внешние приёмники подтверждают позиции после идемпотентной записи; транзакции Kafka–БД нет [^48]|
|Replay: онлайн-проекции| восстановление из changelog/снимка с watermark и хвоста Kafka. Старый архив пересчитывается в отдельную проекцию, чтобы повторно не увеличить действующие счётчики|
|Идемпотентный приёмник: DWH| offset подтверждается после insert_quorum=2. Повтор сохраняет event_id; датасеты и агрегаты читаются с дедупликацией по ID [^49]|
|Saga / checkpoint: медиа и фоновые Job| шаги и попытки сохраняются; повтор продолжает незавершённый шаг. При отмене удаляются незавершённые объекты; Cron не допускает параллельную обработку одной версии|

### 9.2. Цели и проверка восстановления

Проектные цели: для API воспроизведения/рекомендаций — 99,9% технически успешных ответов за 3 с за 30 дней, p95 ≤500 мс; для счётчиков — 99% событий за 60 с.

|Сценарий | Сохранность / цель|
|:--|:--|
|Один core ДЦ| RPO=0 для подтверждённых majority/QUORUM записей MongoDB/Cassandra и ack Kafka при исправных оставшихся копиях. RTO API ≤120 с для контрольных клиентов; Kafka/Streams ≤5 мин [^51]|
|Один API/Ingress-узел| цель переключения ≤30 с. Redis не даёт RPO=0; возможен сброс контекста [^26]|
|Порча данных / потеря журнала| backup → сверка версий/offsets → доступный хвост журнала. Без непрерывного журнала RPO ограничен последней восстановимой точкой|

Ежеквартально: отказ узла/ДЦ, остановка consumer, повтор пачки, восстановление backup. Передача ~1228 ТБ медиа по 10 Гбит/с — минимум ~11,4 суток; полный RTO определяется пробным восстановлением.

## 10. Схема проекта

R/W — чтение/запись, P/C — публикация/чтение событий. Пунктир — Kafka. Синие узлы — API/движки, фиолетовые — фоновые процессы, жёлтые — данные.

```mermaid
%%{init: {"layout": "dagre", "look": "classic", "flowchart": {"curve": "bumpX", "nodeSpacing": 24, "rankSpacing": 48}, "themeVariables": {"fontSize": "13px"}}}%%
flowchart LR
    Client(["Клиент"])
    Front["API Gateway / Composition"]
    CDN["CDN / LFU"]
    Client -->|"API"| Front
    Client -->|"R медиа"| CDN

    subgraph AUTH["Auth"]
        AuthApi["Auth API"]
        AuthDb[("MongoDB auth<br/>AuthIdentity, AuthSession<br/>user_id")]
        Provision["Provision worker"]
        AuthApi -->|"R/W identity, session"| AuthDb
        Provision -->|"R/W PENDING → ACTIVE"| AuthDb
    end

    subgraph USERS["Users"]
        UserApi["Users API"]
        UserDb[("MongoDB users<br/>User")]
        UserApi -->|"R/W профиль"| UserDb
    end
    Provision -->|"API W профиль по user_id"| UserApi

    subgraph CATALOG["Каталог / поиск / медиа"]
        CatalogApi["Catalog / Search API"]
        CatalogDb[("MongoDB catalog<br/>Artist, Album, Track, Media")]
        SearchDb[("OpenSearch")]
        MediaStore[("Ceph: медиа")]
        CatalogCdc["Catalog CDC worker"]
        Indexer["CatalogIndexWorker"]
        MediaWorker["Media publish worker"]
        CatalogApi -->|"R/W метаданные"| CatalogDb
        CatalogApi -->|"R поиск"| SearchDb
        CatalogCdc -->|"R change stream"| CatalogDb
        Indexer -->|"R backfill"| CatalogDb
        Indexer -->|"W индекс / версии"| SearchDb
        MediaWorker -->|"R/W PENDING / READY"| CatalogDb
        MediaWorker -->|"W объекты"| MediaStore
    end
    CatalogApi -->|"API R user_id артиста"| UserApi
    CDN -->|"R промах"| MediaStore

    subgraph REACTIONS["Реакции"]
        ReactionApi["Reactions API"]
        ReactionDb[("Cassandra<br/>Like / Follow")]
        ReactionCdc["Reaction CDC agents"]
        ReactionApi -->|"R/W личное состояние"| ReactionDb
        ReactionCdc -->|"R CDC log"| ReactionDb
    end

    subgraph PLAYBACK["Воспроизведение / история"]
        PlaybackApi["Playback / History API"]
        PlaybackDb[("Cassandra<br/>PlaybackSession / History")]
        QueueDb[("Redis: context / queue")]
        Projector["PlaybackProjector worker"]
        PlaybackApi -->|"R сессии / история"| PlaybackDb
        PlaybackApi -->|"R/W context / cursor"| QueueDb
        Projector -->|"W проекции"| PlaybackDb
    end
    PlaybackApi -->|"API R аудиоверсия"| CatalogApi

    subgraph RECS["Рекомендации / плейлисты"]
        RecApi["Recommendations API<br/>Static / Popular / Followed<br/>merger + RRF"]
        RecDb[("MongoDB recommendations<br/>Playlist.items / FeedDefinition<br/>RecommendationModel")]
        RecCache[("Redis: EngineResultCache<br/>top-100 / TTL 5 мин")]
        Models[("Ceph: модели / CONTENT")]
        CfEngine["BPR / CoListen engines"]
        AnnEngine["ContentANN engine"]
        TrainJob["CollabTrain GPU job"]
        ContentCron["ContentIndex Cron"]
        PopularCron["Popularity Cron"]
        CoListenCron["CoListen Cron"]
        RecCdc["Recommendations CDC"]
        ModelWorker["ModelPublishWorker"]
        RecApi -->|"R/W плейлисты / конфиги"| RecDb
        RecApi -->|"R/W кэш движков"| RecCache
        RecApi -->|"R top-100"| CfEngine
        RecApi -->|"R top-100"| AnnEngine
        CfEngine -->|"R COLLAB"| Models
        AnnEngine -->|"R CONTENT / HNSW"| Models
        CfEngine -->|"R/W кэш"| RecCache
        AnnEngine -->|"R/W кэш"| RecCache
        RecApi -->|"R POPULAR"| Models
        TrainJob -->|"W COLLAB"| Models
        TrainJob -->|"W manifest COLLAB"| RecDb
        ContentCron -->|"W CONTENT / HNSW"| Models
        ContentCron -->|"W READY manifest"| RecDb
        PopularCron -->|"W POPULAR"| Models
        PopularCron -->|"W READY manifest"| RecDb
        CoListenCron -->|"W co-listen"| Models
        CoListenCron -->|"W соседи / manifest"| RecDb
        RecCdc -->|"R change stream"| RecDb
        ModelWorker -->|"R/W active version"| RecDb
        ModelWorker -->|"W reload"| CfEngine
        ModelWorker -->|"W reload"| AnnEngine
    end
    RecApi -->|"API R сигналы"| ReactionApi
    RecApi -->|"API R история"| PlaybackApi
    RecApi -->|"API R метаданные / фильтры"| CatalogApi
    ContentCron -->|"API R bulk metadata"| CatalogApi
    PlaybackApi -->|"API R порция треков"| RecApi

    subgraph EVENTS["Платформа событий / счётчики"]
        IngestApi["Event Ingest API"]
        Kafka[("Kafka")]
        CounterWorker["CounterAggregator worker"]
        CounterDb[("Redis: CounterReadModel")]
        CounterApi["Counters Read API"]
        SnapshotCron["Snapshot Cron"]
        IngestApi -.->|"P heartbeat / finish"| Kafka
        Kafka -.->|"C reactions / counted / control"| CounterWorker
        CounterWorker -->|"W абсолютные итоги"| CounterDb
        CounterApi -->|"R готовые итоги"| CounterDb
        SnapshotCron -.->|"P snapshot request"| Kafka
        CounterWorker -.->|"P CounterUpdated / Snapshot"| Kafka
    end
    PlaybackApi -.->|"P start, event_id от request_id"| Kafka
    CatalogCdc -.->|"P CatalogChanged"| Kafka
    ReactionCdc -.->|"P ReactionChanged"| Kafka
    RecCdc -.->|"P ModelReady / config"| Kafka
    Kafka -.->|"C CatalogChanged / CounterUpdated"| Indexer
    Kafka -.->|"C playback / reactions"| Projector
    Projector -.->|"P TrackPlayCounted"| Kafka
    Kafka -.->|"C ModelReady"| ModelWorker

    subgraph ANALYTICS["DWH / датасеты"]
        DwhWriter["DWHWriter worker"]
        Dwh[("ClickHouse<br/>DomainEvent / CounterSnapshot")]
        DatasetApi["Dataset API"]
        ExportCron["FeatureExport Cron"]
        Features[("Ceph: датасеты")]
        DwhWriter -->|"W архив / снимки"| Dwh
        ExportCron -->|"R признаки"| Dwh
        ExportCron -->|"W snapshot"| Features
        DatasetApi -->|"R snapshot"| Features
        DatasetApi -->|"R агрегаты"| Dwh
    end
    Kafka -.->|"C бизнес-события"| DwhWriter
    TrainJob -->|"API R training snapshot"| DatasetApi
    PopularCron -->|"API R top за 24 часа"| DatasetApi
    CoListenCron -->|"API R co-listen пары"| DatasetApi

    Front -->|"R/W доступ"| AuthApi
    Front -->|"R/W профиль"| UserApi
    Front -->|"R каталог / поиск"| CatalogApi
    Front -->|"R/W реакции"| ReactionApi
    Front -->|"R/W воспроизведение"| PlaybackApi
    Front -->|"R/W подборки"| RecApi
    Front -->|"W телеметрия"| IngestApi
    Front -->|"R счётчики"| CounterApi

    classDef api fill:#deebff,stroke:#527bbd
    classDef worker fill:#eee0ff,stroke:#8a63b8
    classDef db fill:#fff0cf,stroke:#c58a25
    classDef entry fill:#e2efd5,stroke:#729857
    class AuthApi,UserApi,CatalogApi,ReactionApi,PlaybackApi,RecApi,CounterApi,IngestApi,DatasetApi,CfEngine,AnnEngine api
    class Provision,CatalogCdc,Indexer,MediaWorker,ReactionCdc,Projector,TrainJob,ContentCron,PopularCron,CoListenCron,RecCdc,ModelWorker,CounterWorker,SnapshotCron,DwhWriter,ExportCron worker
    class AuthDb,UserDb,CatalogDb,SearchDb,MediaStore,ReactionDb,PlaybackDb,QueueDb,RecDb,RecCache,Models,Kafka,CounterDb,Dwh,Features db
    class Client,Front,CDN entry
```

MongoDB — разные БД/ACL, Redis — префиксы/ACL, Ceph — бакеты доменов.

## 11. Список серверов

Размещение по [лекции 11](https://github.com/init/highload/blob/main/highload_l11_hosting.md): аренда bare-metal, API/Ingress в Kubernetes, часть БД в KVM-VM, обучение на GPU.

### 11.1. Хостинг и оборудование

|Провайдер | Размещение | Причина выбора|
|:--|:--|:--|
|NETRACK| три core ДЦ: IXcellerate North, IXcellerate South, StoreData| отдельные московские площадки, аренда серверов, частные сети и BGP [^32]|
|EdgeCenter| собственный LFU-кэш в Москве, Петербурге, Екатеринбурге, Новосибирске| bare-metal в нужных регионах; управляемый CDN не используется [^33]|
|Selectel| отдельная площадка в Петербурге для бэкапов| независимость от рабочих кластеров и их учётных данных [^34]|
|HOSTKEY| GPU-пул в одном московском ДЦ, вне онлайн-контура| конфигурация класса gpu.v4-6000Pro-10g; наличие пяти узлов и private VLAN требуют КП [^37]|

Смета на 19.11.2026 с НДС; индивидуальные ставки требуют КП [^35]. Аренда включает размещение, питание и замену оборудования; сеть оплачивается отдельно. Покупка — оценка полного сервера [^38].

CPU — физические ядра x86-64, SMT отключён; GPU-тариф — в vCPU. RAM — ГиБ, диски — десятичные ТБ. Core-хосты поровну по трём ДЦ. Реплики и резерв включены; VM и контейнеры отдельно не считаются.

|Назначение / сервисы | CPU / RAM на сервер | Диски на сервер | Сеть | Количество | Аренда, тыс. ₽/шт·мес | Аренда, тыс. ₽/мес | Покупка, млн ₽/шт|
|:--|:--|:--|:--|--:|--:|--:|--:|
|K8s workers: API, Ingress, обработчики, RGW, эксплуатация| 64 / 256| 2×1,92 NVMe, RAID1| 2×25 Гбит/с| 18 (6/ДЦ)| 70| 1260| 1,85|
|K8s control plane / etcd| 8 / 32| 2×0,48 SSD, RAID1| 2×10 Гбит/с| 9 (3/ДЦ)| 12| 108| 0,5|
|Координаторы: MongoDB config server, Keeper, KRaft, Ceph MON/MGR, OpenSearch managers| 16 / 64| 2×0,96 SSD, RAID1| 2×10 Гбит/с| 3 (1/ДЦ)| 25| 75| 0,65|
|Общие KVM-хосты: два MongoDB-шарда| 80 / 768| 8×3,84 NVMe: отдельные RAID10 по 4 диска; 2×0,96 SSD под ОС| 2×25 Гбит/с| 3| 110| 330| 2,7|
|Cassandra| 32 / 128| 4×1,92 NVMe, JBOD| 2×25 Гбит/с| 9| 45| 405| 1,3|
|Redis: 6 primary + 6 replica| 32 / 512| 2×1,92 NVMe, RAID1| 2×25 Гбит/с| 12| 65| 780| 1,4|
|Общие KVM-хосты: Kafka broker + OpenSearch data node| 48 / 256| Kafka: 4×3,84 NVMe, JBOD; поиск: 2×3,84 NVMe, RAID1; 2×0,96 SSD под ОС| 2×25 Гбит/с| 6| 65| 390| 1,55|
|ClickHouse: 3 шарда × 3 реплики| 32 / 256| 12×16 HDD, RAID10 + 2×0,96 SSD под ОС| 2×25 Гбит/с| 9| 95| 855| 2,2|
|Ceph OSD| 32 / 128| 24×20 HDD, JBOD + 2×0,96 SSD под ОС + 2×3,84 NVMe, RAID1 под DB/WAL| 2×25 Гбит/с| 12| 125| 1500| 4|
|Rec-engine hosts: BPR/CoListen, ContentANN, CPU jobs| 64 / 512| 4×3,84 NVMe, JBOD| 2×25 Гбит/с| 3| 95| 285| 2,5|
|GPU train: 1×RTX PRO 6000 Blackwell, 96 ГБ VRAM на хост| 64 vCPU / 1024| 2×3,84 NVMe| 10 Гбит/с; private VLAN| 5 (4 рабочих + 1 резерв)| 230| 1150| 6|
|CDN: HTTP-кэш LFU| 16 / 64| 2×3,84 NVMe под кэш| 2×10 Гбит/с| 25 (10/7/4/4)| 35| 875| 0,8|
|CDN: LVS / keepalived| 16 / 32| 2×0,48 SSD, RAID1| 2×100 Гбит/с| 8 (2/площадку)| 30| 240| 0,7|
|Бэкапы: отдельный репозиторий| 16 / 128| 24×20 HDD, RAID6 + 2×0,96 SSD под ОС| 2×10 Гбит/с| 6| 100| 600| 3,5|
|**Итого**| | | | **128**| | **8853**| **237,55 всего**|

На хосте координаторов — по экземпляру каждого компонента, до 2 CPU / 4 ГиБ на процесс. На MongoDB-хосте — по одной VM каждого шарда.

**Проверка ёмкости** (не более 70% заполнения):

|Хранилище | Доступно с учётом копий / резерва | Требуется|
|:--|:--|:--|
|Ceph| 12 × 24 × 20 × 0,7 / 3 = **1344 ТБ**| 1228,3 ТБ медиа + ~62 ТБ/год + модели до 2 ТБ и датасеты до 5 ТБ|
|MongoDB| 2 шарда × 7,68 × 0,7 = **10,75 ТБ**| Auth/Users/Catalog/плейлисты/конфиги: бюджет до 5 ТБ с индексами|
|Cassandra| 9 × 7,68 × 0,7 / 3 = **16,13 ТБ**| история, реакции, сессии: бюджет до 8 ТБ с накладными расходами|
|Redis| 6 primary × 512 × 0,7 = **2150 ГиБ**| очереди, engine cache и счётчики: бюджет до 1 ТБ с накладными расходами|
|Kafka| 6 × 15,36 × 0,7 / 3 = **21,50 ТБ**| ~2,08 ТБ событий за 7 суток + changelog и накладные расходы|
|OpenSearch| 6 × 3,84 × 0,7 / 3 = **5,38 ТБ**| проектный индекс до 1 ТБ; проверяется на 85 млн документов|
|ClickHouse| 3 шарда × 96 × 0,7 = **201,6 ТБ**| playback-DWH ~108,6 ТБ/год плюс производные события/снимки; бюджет до 180 ТБ|
|Бэкапы| 6 × 22 × 20 × 0,7 = **1848 ТБ** после RAID6| одна полная копия медиа/БД/DWH + 30 суток инкрементов, до 1600 ТБ|

### 11.2. Контейнеры и VM

Количество — **на ДЦ**, кроме суммарных значений. r/l — requests/limits; через «/» указаны отдельные компоненты. Общие Cron и GitLab учтены в бюджете одного ДЦ.

|Контейнер | Количество | CPU r/l | RAM r/l, ГиБ | Пул|
|:--|:--|:--|:--|:--|
|Auth / Users / Reactions API| 2 / 2 / 2| 2 / 4| 4 / 8| workers|
|Catalog / Event Ingest / Recommendations API| 4 / 8 / 3| 4 / 8| 8 / 16| workers|
|Playback / History API| 2| 2 / 4| 4 / 8| workers|
|Counters Read API| 2| 2 / 4| 4 / 8| workers|
|Dataset API| 2| 1 / 2| 2 / 4| workers|
|Gateway / Composition| 3| 2 / 4| 4 / 8| workers|
|Catalog CDC / Recommendations CDC| 1 / 1| 2 / 4| 4 / 8| workers|
|Provision worker / ModelPublishWorker| 1 / 1| 1 / 2| 2 / 4| workers|
|PlaybackProjector / CounterAggregator| 2 / 2| 4 / 8| 8 / 16| workers|
|DWHWriter| 2| 4 / 8| 8 / 16| workers|
|CatalogIndexWorker| 2| 2 / 4| 4 / 8| workers|
|Media publish worker| 2| 2 / 4| 4 / 8| workers|
|mongos| 2| 2 / 4| 4 / 8| workers|
|Ceph RGW| 2| 4 / 8| 8 / 16| workers|
|Snapshot Cron| 1 всего| 2 / 4| 4 / 8| workers|
|FeatureExport Cron| 1 всего| 2 / 4| 8 / 16| workers|
|CoreDNS| 2| 0,5 / 1| 0,5 / 1| workers|
|Prometheus| 2| 2 / 4| 8 / 16| workers|
|Alertmanager| 1| 0,5 / 1| 1 / 2| workers|
|Grafana / ArgoCD| 1 / 1| 1 / 2| 2 / 4| workers|
|Loki| 1| 2 / 4| 8 / 16| workers|
|CI runner| 1| 4 / 8| 8 / 16| workers|
|GitLab / Registry| 1 всего| 4 / 8| 8 / 16| workers|
|Fluent Bit| 1/узел, 30 всего| 0,25 / 0,5| 0,5 / 1| все K8s-пулы|
|kube-proxy / CNI| по 1/узел, по 30 всего| 0,25 / 0,5| 0,25 / 0,5| все K8s-пулы|
|NGINX Ingress| 4| 8 / 12| 8 / 16| workers, разные узлы|
|Envoy у worker API / Gateway| 30| 0,25 / 0,5| 0,5 / 1| workers|
|kube-apiserver| 3| 1 / 2| 2 / 4| control plane|
|etcd Kubernetes| 3| 1 / 2| 4 / 8| control plane|
|kube-controller-manager| 3| 0,5 / 1| 1 / 2| control plane|
|kube-scheduler| 3| 0,5 / 1| 0,5 / 1| control plane|
|BPR / CoListen engine, один сервис| 1| 16 / 24| 192 / 256| Rec-engine|
|ContentANN engine| 1| 8 / 12| 96 / 128| Rec-engine|
|CoListen / Popularity / ContentIndex Cron, отдельные Job| суммарно ≤1 heavy Job/ДЦ| 16 / 20| 64 / 96| Rec-engine, CPU batch quota|
|Cassandra CDC agent, Docker на Cassandra| 1/узел, 9 всего| — / 1| — / 2| вне Kubernetes|
|BPR train, Docker / TorchRec| 4 рабочих + 1 резерв| — / 56 vCPU| — / 896| GPU-host; 1 GPU на контейнер|

После резерва ОС/kubelet worker даёт 60 CPU / 240 ГиБ, шесть — **360 CPU / 1440 ГиБ на ДЦ**. Requests: **203,5 CPU / 387 ГиБ**. При потере ДЦ API (кроме Dataset), projector/aggregator, DWHWriter, Indexer, media/RGW и mongos масштабируются ×1,5 с округлением вверх; Envoy — до 45. Итого **275,25 CPU / 530,5 ГиБ**, что помещается на 5 workers (300 CPU / 1200 ГиБ). Ingress — на разных узлах, CI — низкий приоритет.

BPR/CoListen + ContentANN + один CPU Job: 40 CPU / 352 ГиБ requests, 56 CPU / 480 ГиБ limits, плюс системные агенты. Факторы и co-listen — ~144 ГБ до ID-индексов, в packed arrays/mmap. Модели обновляются по ДЦ без двойной загрузки движка.

Плановые RPS на pod: Gateway — 5000, Event Ingest / Playback / Counters Read — 2000, Catalog — 1000, прочие API — 500. Проверяются нагрузкой с внутренними вызовами и password hashing. Постоянные тома — Ceph; логи после семплирования — до 100 ГБ/сутки, 7 суток.

VM на общих хостах; vCPU привязаны к физическим ядрам, без overcommit:

|VM / сервис | VM на ДЦ | vCPU / RAM, ГиБ на VM | Диски / изоляция|
|:--|--:|:--|:--|
|MongoDB shard A replica| 1| 32 / 256| свои 4×3,84 NVMe, RAID10|
|MongoDB shard B replica| 1| 32 / 256| свои 4×3,84 NVMe, RAID10|
|Kafka broker| 2| 16 / 64| свои 4×3,84 NVMe, JBOD|
|OpenSearch data node| 2| 16 / 128| свои 2×3,84 NVMe, RAID1|

MongoDB VM занимают 64 из 80 CPU и 512 из 768 ГиБ; Kafka + OpenSearch — 32 из 48 CPU и 192 из 256 ГиБ. Остаток — гипервизор и резерв. libvirt задаёт CPU/RAM и I/O-лимиты [^39]; отказ общего хоста затрагивает оба кластера.

### 11.3. Месячная стоимость

Бюджет сети: ~27 ПБ/месяц внешнего трафика, до ~2,8 ПБ origin-трафика. Полоса и модель без оплаты каждого ГБ согласуются в КП [^36]. Обмен признаками с GPU — до 10 ТБ/месяц; распределённое обучение использует отдельную внутреннюю сеть.

|Статья | Расчёт | тыс. ₽/мес|
|:--|:--|--:|
|Серверы с размещением и питанием| таблица 11.1, включая 5 GPU-хостов| 8853|
|Внешние каналы CDN| 4 × 100 Гбит/с × 500 тыс. ₽| 2000|
|Внешние каналы core| 3 × 25 Гбит/с × 150 тыс. ₽| 450|
|Меж-ДЦ сеть| 3 связи по 2×25 Гбит/с, независимые трассы; 200 тыс. ₽/связь| 600|
|Канал бэкапов| 10 Гбит/с| 100|
|DNS, адреса, BGP и дополнительные подключения| общий бюджет| 50|
|Внутренняя сеть GPU| 5 портов 10 Гбит/с / VLAN, сметно| 100|
|**Итого инфраструктура**| | **12 153**|
|Резерв сметы 20%| 12 153 × 0,2| 2430,6|
|**Плановый месячный бюджет**| | **14 583,6 (~14,58 млн ₽)**|

Смета без зарплат, музыкальных лицензий и разовых работ. Покупка серверов не включает размещение, питание, каналы и внешнее сетевое оборудование. CDN резервируется по узлам, без покрытия потери целой площадки.

### 11.4. GPU для рекомендаций

GPU обучают BPR-MF; онлайн-движки работают на CPU.

**Память:** (30,5 млн пользователей + 85 млн треков) × 128 × 4 = **59,14 ГБ параметров FP32**. TorchRec шардирует таблицы между GPU [^40]; SGD — с разреженными градиентами без momentum [^37].

**Допущение:** ежедневно обучаем по 7 суткам данных, 3 эпохи за 8 часов: 186 млн × 7 × 3 = **3,906 млрд примеров**. Пример — положительное взаимодействие с выбранным отрицательным.

Рабочие GPU: G = ceil(3,906 млрд / (q × 28 800 × η)), где q — примеров/с на GPU, η = 0,7 — коэффициент загрузки/масштабирования:

|Предполагаемая q, примеров/с на GPU | Рабочих GPU | С резервом одного хоста|
|--:|--:|--:|
|200 000| 1| 2|
|100 000| 2| 3|
|50 000| 4| 5|
|25 000| 8| 9|

**Смета: 4 рабочих + 1 резервный GPU-хост** при q ≥50 тыс./с и η ≥0,7; значения требуют бенчмарка. Для одного GPU нужно q ≥193 750 примеров/с. Отдельно проверяются подготовка признаков и построение HNSW.

GPU-тариф — предзаказ [^37]; наличие пяти совместимых узлов на одной площадке и private VLAN требует КП.

При отказе GPU обучение перезапускается на резервном хосте из checkpoint; онлайн использует предыдущую модель.

### 11.5. Сравнение размещения

Сравнение при одинаковом количестве реплик БД:

|Объединение | Раздельно, шт. / тыс. ₽·мес | Вместе, шт. / тыс. ₽·мес | Экономия, тыс. ₽/мес|
|:--|:--|:--|--:|
|Workers + Ingress| 27×32 CPU/128 ГиБ + 12×16 CPU/64 ГиБ; **39 / 1515**| 18×64 CPU/256 ГиБ; **18 / 1260**| 255|
|Два MongoDB-шарда| 6×32 CPU/256 ГиБ; **6 / 390**| 3×80 CPU/768 ГиБ; **3 / 330**| 60|
|Kafka + OpenSearch| 6×16 CPU/64 ГиБ + 6×16 CPU/128 ГиБ; **12 / 480**| 6×48 CPU/256 ГиБ; **6 / 390**| 90|
|**Итого**| **57 / 2385**| **27 / 1980**| **405**|

Ceph/ClickHouse оставлены отдельно из-за дисковой нагрузки, Redis — RAM и размещения реплик; ML, control plane и координаторы — отдельные пулы.

|Вариант | Физических серверов | Аренда серверов, млн ₽/мес | Сеть + аренда + резерв 20%, млн ₽/мес | Покупка серверов, млн ₽|
|:--|--:|--:|--:|--:|
|Исходный, без GPU| 153| 8,108| 13,570| 217,15|
|Раздельное размещение + 5 GPU-хостов| 158| 9,258| 15,070| 247,15|
|**Совместное размещение + 5 GPU-хостов, бюджетный сценарий**| **128**| **8,853**| **14,584**| **237,55**|

Экономия аренды — **405 тыс. ₽/мес (~4,86 млн ₽/год)**, с резервом сметы — **486 тыс. ₽/мес**, покупки — **9,6 млн ₽**. Исходные ставки покупки: workers — 1,2; Ingress — 0,7; MongoDB — 1,5; Kafka — 0,9; OpenSearch — 0,85 млн ₽/шт.

Совместное размещение выгодно при аренде хоста до ~84,2 тыс. ₽/мес для workers, 130 тыс. для MongoDB, 80 тыс. для Kafka/OpenSearch. Сметные ставки — 70/110/65 тыс. ₽; проверяются КП, совместная нагрузка, p99, I/O и восстановление обоих кластеров.

[^1]: [Аудитория «Яндекс Музыки» превысила 30,5 млн человек в месяц](https://www.sostav.ru/publication/auditoriya-yandeks-muzyki-prevysila-30-5-mln-chelovek-v-mesyats-79308.html)
[^2]: [Число подписчиков Яндекс Плюса достигло 30 миллионов](https://yandex.ru/company/news/03-07-02-2024)
[^3]: [РОМИР: 85% жителей городов России пользуются музыкальными стриминговыми сервисами](https://romir.ru/feed/romir-85-jiteley-gorodov-rossii-polzuyutsya-muzykalnymi-strimingovymi-servisami)
[^4]: [Средняя длина песен с годами сокращается](https://www.kommersant.ru/doc/5170250) (Коммерсантъ, со ссылкой на исследование UCLA)
[^5]: [Более 4000 единиц контента удалены с «Яндекс Музыки» из-за «фейков» и ЛГБТ-пропаганды](https://www.forbes.ru/forbeslife/499638-bolee-4000-edinic-kontenta-udaleny-s-andeks-muzyki-iz-za-fejkov-i-lgbt-propagandy) (Forbes.ru — цифра «более 76 млн треков», 2023)
[^6]: [Transparency Report | Яндекс Музыка](https://yandex.ru/support/music/ru/rules/transparencyreport)
[^7]: [Качество звука | Яндекс Музыка](https://yandex.ru/support/music/ru/listening/quality)
[^8]: [MSK-IX вступает в клуб «Т»](https://www.msk-ix.ru/news/129/) (официальная новость MSK-IX о пиковой нагрузке 8,7 Тбит/с и топ-5 в мире)
[^9]: [Почему 69% россиян живут в европейской части России](https://www.ixbt.com/live/travel/pochemu-69-rossiyan-zhivut-v-evropeyskoy-chasti-rossii-kotoraya-zanimaet-lish-20-territorii-strany.html) (со ссылкой на данные Росстата)
[^10]: [«Яндекс» построит новый дата-центр во Владимирской области](https://servernews.ru/1129768)
[^11]: [What is Anycast? How does Anycast Work?](https://www.cloudflare.com/learning/cdn/glossary/anycast-network/) (Cloudflare Learning Center)
[^16]: [Testing the Performance of NGINX and NGINX Plus Web Servers](https://blog.nginx.org/blog/testing-the-performance-of-nginx-and-nginx-plus-web-servers) (nginx, 2017, CPS для HTTPS-запросов)
[^17]: [Testing the Performance of NGINX Ingress Controller for Kubernetes](https://blog.nginx.org/blog/testing-performance-nginx-ingress-controller-kubernetes) (nginx, 2019, TPS)
[^18]: [Testing TLS Session Resumption](http://vincent.bernat.im/en/blog/2011-ssl-session-reuse-rfc5077.html) (Vincent Bernat, сбор статистики по использованию SSL session tickets через tcpdump)
[^19]: [NGINX on Kubernetes](https://kubernetes.nginx.org/) — различие поддерживаемого NGINX Ingress Controller и завершившего поддержку community ingress-nginx; [настройка балансировки](https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/configmap-resource/).
[^20]: [Kubernetes: Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/) — роль kube-proxy, режим nftables и устаревание режима IPVS с версии 1.35.
[^21]: [ClickHouse Keeper](https://presentations.clickhouse.com/meetup54/keeper.pdf) — материал разработчиков о режимах запуска и координации.
[^22]: [Varnish: Configuring Varnish](https://docs.varnish-software.com/book/operations/configuring-varnish/) — штатное вытеснение LRU при нехватке места.
[^23]: [hnswlib](https://github.com/nmslib/hnswlib) — инкрементальное построение HNSW; [Annoy](https://github.com/spotify/annoy) — статический индекс после build.
[^24]: [OAuth 2.0, RFC 6749](https://www.rfc-editor.org/rfc/rfc6749), [JWT, RFC 7519](https://www.rfc-editor.org/rfc/rfc7519), [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html) — разграничение авторизации, формата токена и аутентификации.
[^25]: [Prometheus: Overview](https://prometheus.io/docs/introduction/overview/) — сбор метрик, визуализация и обработка оповещений.
[^26]: [Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) — hash slots, репликация, failover; отдельные логические БД Redis Cluster не поддерживает.

[^27]: [Apache Kafka: Design](https://kafka.apache.org/design/) — порядок в партициях, транзакции и границы exactly-once при работе с внешними хранилищами.
[^28]: [Apache Cassandra: Change Data Capture](https://cassandra.apache.org/doc/stable/cassandra/managing/operating/cdc.html) — чтение сохранённых мутаций, необходимость потребителя и ограничение CDC-пространства.
[^29]: [OpenSearch: Text analysis](https://docs.opensearch.org/latest/analyzers/) — анализ текстовых полей и инвертированный индекс.

[^30]: [MongoDB: Replication](https://www.mongodb.com/docs/manual/replication/) — replica set, primary и выборы; в каждом шарде по одной копии на ДЦ.
[^32]: [NETRACK: дата-центры](https://netrack.ru/data-centers), [аренда серверов](https://netrack.ru/dedicated) и [сетевая связность](https://netrack.ru/add-services/internet). Размещение на конкретных площадках и конфигурации согласуются до заказа.
[^33]: [EdgeCenter: Bare Metal](https://edgecenter.ru/knowledge-base/cloud/about-bare-metal-severs) — Москва, Петербург, Екатеринбург и Новосибирск; индивидуальная комплектация и сеть — по согласованию.
[^34]: [Selectel: выделенные серверы](https://selectel.ru/services/dedicated/) — площадка для независимого репозитория резервных копий в Петербурге.
[^35]: [HOSTKEY: открытые конфигурации и цены](https://hostkey.ru/documentation/server_order/bm_list/) — рыночный ориентир, просмотрено 15.11.2026; ставки таблицы 11.1 являются проектной сметой, а не переносом тарифов HOSTKEY на других провайдеров.
[^36]: [NETRACK: интернет-порты](https://netrack.ru/add-services/internet) — стоимость высоких скоростей по запросу; [Selectel: модели учёта трафика](https://docs.selectel.ru/dedicated/networks/traffic-plans/) — включённый трафик ограничен выбранной моделью, расширение оплачивается отдельно.
[^37]: [HOSTKEY: GPU-серверы](https://hostkey.ru/gpu-dedicated-servers/) — gpu.v4-6000Pro-10g: 64 vCPU, 1024 ГБ RAM, 2×3,84 ТБ NVMe, 1×6000 PRO 96 ГБ, 10 Гбит/с / 50 ТБ за 230 тыс. ₽/мес, предзаказ, просмотрено 19.11.2026. Наличие в Москве и private VLAN требуют КП. [NVIDIA RTX PRO 6000 Blackwell](https://www.nvidia.com/en-us/products/workstations/professional-desktop-gpus/rtx-pro-6000-family/) — 96 ГБ VRAM. [PyTorch Embedding](https://docs.pytorch.org/docs/stable/generated/torch.nn.Embedding.html) — sparse gradients и совместимые оптимизаторы.
[^38]: Покупка — проектная оценка, не КП на 128 серверов. [ServerFlow: RTX PRO 6000 Blackwell Server Edition 96 ГБ](https://serverflow.ru/catalog/komplektuyushchie/videokarty/videokarta-nvidia-rtx-pro-6000-blackwell-server-edition-96gb-gddr7/) — опубликовано ~3,6 млн ₽ за карту под заказ; 6 млн ₽ на GPU-сервер включает платформу с 1 ТБ RAM, CPU, NVMe и сетью; это сметная оценка. [KNS: HDD 20 ТБ](https://www.kns.ru/product/zhestkii-disk-seagate-exos-x20-20tb-st20000nm007d/) — ориентир последней цены ~111 тыс. ₽; наличие и оптовые условия уточняются.
[^39]: [libvirt: Domain XML](https://libvirt.org/formatdomain.html) — выделение vCPU, памяти и настройка дискового I/O для KVM-VM.

[^40]: [TorchRec](https://meta-pytorch.org/torchrec/overview.html) — шардирование таблиц эмбеддингов и распределённое обучение; производительность зависит от плана размещения и межсоединения.
[^42]: [MongoDB TTL](https://www.mongodb.com/docs/manual/core/index-ttl/) — очистка просроченных документов выполняется фоново; Auth API проверяет срок и отзыв независимо от удаления. JWT/OIDC — источники [^24].
[^43]: [MongoDB: atomicity](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/), [shard keys](https://www.mongodb.com/docs/manual/core/sharding-shard-key/) и [document limit](https://www.mongodb.com/docs/manual/core/document/) — атомарность документа с expected revision, hashed sharding, предел BSON 16 MiB.
[^44]: [Database per service](https://microservices.io/patterns/data/database-per-service.html) — приватные данные владельца, доступ через API; независимость схем не требует отдельной физической машины для каждой.
[^45]: [BPR: Bayesian Personalized Ranking](https://arxiv.org/abs/1205.2618) — оптимизация персонального порядка по implicit feedback; в проекте выбран вариант матричной факторизации.
[^46]: [TruncatedSVD](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.TruncatedSVD.html) и [TF-IDF](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html) — контентный baseline по метаданным; качество и параметры проверяются отдельно.
[^47]: [Cormack, Clarke, Büttcher: Reciprocal Rank Fusion](https://cormack.uwaterloo.ca/cormack/cormacksigir09-rrf.pdf) — объединение рангов; веса движков в проекте задаются FeedDefinition.
[^48]: [Kafka Streams: processing.guarantee](https://kafka.apache.org/41/streams/developer-guide/config-streams/#processing-guarantee) — режим exactly_once_v2; границы транзакций с внешними хранилищами — [^27].
[^49]: [ClickHouse: Replicated table engines](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication) — репликация и подтверждение вставки через insert_quorum.
[^51]: [MongoDB: Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/) и [Cassandra: Dynamo](https://cassandra.apache.org/doc/stable/cassandra/architecture/dynamo.html) — подтверждение записи большинством реплик и уровни согласованности.
