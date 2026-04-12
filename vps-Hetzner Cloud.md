**Полная инструкция по Hetzner Cloud с самого начала — одним текстом. Копируй и выполняй по шагам.**

Открой браузер и зайди на https://www.hetzner.com/cloud (или сразу https://console.hetzner.cloud/).  
В правом верхнем углу нажми **Register now** (или «Зарегистрироваться»).  
Введи email, придумай пароль, заполни личные данные (имя, фамилия, адрес — точно как в паспорте/выписке), страну **Cyprus**.  
Добавь способ оплаты (кредитная/дебетовая карта или PayPal) — это обязательно, но деньги спишутся только после создания сервера (почасовая тарификация).  
Иногда просят верификацию по паспорту/ID — загрузи фото (процесс занимает 5–30 минут).  
После подтверждения email зайди в консоль: https://console.hetzner.cloud/.

**Создаём проект**  
В левом меню нажми **Projects** → **Create project**. Назови его `crypto-collector`.

**Добавляем SSH-ключ** (один раз)  
В левом меню **Security → SSH Keys → Add SSH Key**.  
На своём компьютере выполни команду:  
`ssh-keygen -t ed25519 -C "твой_email@example.com"`  
(нажми Enter три раза — ключ создастся в `~/.ssh/id_ed25519.pub`).  
Открой файл `id_ed25519.pub` (блокнот) и скопируй весь текст.  
Вставь его в Hetzner (Name: `my-key`), нажми **Add Key**.

**Создаём сервер**  
В проекте нажми **Create Resource → Server**.  
Заполни:  
- **Location**: сначала попробуй **Singapore** (лучший пинг к Binance/Bybit/OKX). Если нет — выбирай **Nuremberg (NBG)** или **Falkenstein (FSN)**.  
- **Image**: **Ubuntu 24.04** (стандартная).  
- **Type**: **Cost-Optimized** → **CX22** (2 vCPU, 4 GB RAM, 40 GB NVMe) — идеально и очень дёшево (€4–8/мес). Если хочешь чуть быстрее — **CPX22** или **CAX21** (ARM).  
- **SSH Keys**: поставь галочку напротив твоего `my-key`.  
- **Networking**: оставь по умолчанию (Public IPv4).  
- **Name**: `crypto-collector`.  
Нажми **Create** (стоимость ≈ €0.012 в час — меньше 10 евро в месяц).  

Через 30–60 секунд сервер будет **Running**. Запиши **IPv4-адрес** (появится в списке серверов).

**Подключаемся первый раз по SSH**  
На компьютере выполни:  
`ssh -i ~/.ssh/id_ed25519 ubuntu@ТВОЙ_IP`  
(На Windows используй Windows Terminal или PuTTY).  
Если зашёл — ты внутри (prompt `ubuntu@crypto-collector`).

Выполни по очереди эти команды:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-venv python3-pip git tmux curl unzip
mkdir -p ~/collector && cd ~/collector
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install websockets pyarrow pandas numpy tqdm nest_asyncio plotly ipywidgets jupyter aiofiles
```

Сервер полностью готов.

**Настраиваем VS Code + Remote-SSH + Live Server** (самый удобный способ)

На своём ноутбуке:  
1. Установи **VS Code** (если нет).  
2. Extensions (Ctrl+Shift+X) → установи:  
   - Remote - SSH (Microsoft)  
   - Python (Microsoft)  
   - Jupyter (Microsoft)  
   - Live Server (Ritwick Dey)  
3. Перезапусти VS Code.

В VS Code нажми внизу слева **><** (Remote) → **Connect to Host... → Add New SSH Host...**  
Вставь одну строку:  
`ssh -i ~/.ssh/id_ed25519 ubuntu@ТВОЙ_IP`  
Сохрани хост под именем `hetzner-collector`.  
Подключись (Remote Explorer → твой хост → Connect → Linux).  

Теперь внизу слева написано **SSH: hetzner-collector** — ты работаешь на сервере как на своём компьютере.

В терминале VS Code (Ctrl+`) выполни:

```bash
cd ~/collector
source venv/bin/activate
```

Создай три файла в папке `~/collector`:  
- `Notebook1_Collector.ipynb`  
- `Notebook2_Analysis.ipynb`  
- `Notebook3_Visualizer.ipynb`

**Как работать дальше**

- **Сбор данных (Collector)**:  
  `tmux new -s collector`  
  `source venv/bin/activate`  
  `python collector.py --collect 14400`  
  (Ctrl+B → D — отсоединиться. Вернуться: `tmux attach -t collector`)

- **Анализ**: открывай .ipynb файлы в VS Code → **Select Kernel** → `~/collector/venv/bin/python` → запускай ячейки.

- **Визуализатор (explorer.html)**: правой кнопкой по файлу `explorer.html` → **Open with Live Server**. Браузер откроется автоматически со всеми графиками.

Collector работает 24/7 даже если закроешь VS Code или ноутбук. Оплата почасовая — можно удалить сервер в любой момент.

Как только подключишься через Remote-SSH и увидишь папку `~/collector`, напиши мне **«Я внутри VS Code»** — я сразу пришлю тебе **готовый collector.py** (полностью собранный из твоего slave_plan1.md под все 12 бирж) и шаблоны трёх ноутбуков.

Всё настроено за 15–20 минут. Стабильный интернет, никаких обрывов WS. Поехали! 🚀