# low-latency-lab — снижение задержки при работе через Tailscale + ThinLinc

> Набор инструкций и твиков для улучшения отзывчивости удалённого рабочего стола.

---

### 1. Убедитесь, что Tailscale использует **прямое соединение,** а не через **DERP**

Tailscale по умолчанию использует **DERP** (Relay серверы), чтобы временно связать два узла, пока не будет найден прямой маршрут (P2P). Если P2P-соединение не удаётся установить (из-за строгого NAT, блокировки UDP и т.д.), соединение остаётся через DERP, что значительно увеличивает задержку и нестабильность.

```bash
tailscale status         # должно быть "direct", а не "relay"
tailscale ping <host>    # проверка доступности узла
tailscale netcheck       # оценка NAT, UDP, DERP, P2P
```

#### Поддержка P2P соединения

- Tailscale **всегда стремится** установить прямой UDP-туннель между узлами.
- **DERP** подключение используется только когда прямой маршрут невозможен.
- Для поддержки P2P:
  - Откройте или пробросьте **UDP порт 41641** на публичном IP
  - Включите в роутере **UPnP** или **NAT-PMP**

---

### 2. Настройка ThinLinc для более плавной работы

В клиенте ThinLinc (вкладка **Optimization**) можно настроить поведение VNC-протокола под качество сети:

- **Auto Select** — режим по умолчанию:
  - ThinLinc автоматически выбирает кодек и параметры в зависимости от текущей пропускной способности и задержки.

- **Preferred encoding** — ручной выбор кодека:
  - `Tight` — сочетает zlib и JPEG, даёт минимальный трафик, но высокую нагрузку на CPU. Рекомендуется для слабых сетей.
  - `Hextile`, `ZRLE`, `Raw` — передают больше данных, но быстрее работают на хорошей сети.

- **Compression level (0–9)** — степень общего сжатия (zlib):
  - `2–3` — на хорошей сети
  - `7–9` — при медленном соединении и высокой задержке

- **JPEG compression** — степень JPEG-сжатия цветных областей:
  - `2–3` — на хорошей сети
  - `7–9` — при медленном соединении и высокой задержке

- **Color level** — глубина цвета:
  - `Full` — высокая детализация, большой объём данных
  - `Medium` или `Low` — снижают нагрузку на канал, подходят для медленных сетей

- **SSH Compression** — по умолчанию выключена:
  - Не рекомендуется включать, т.к. графика уже эффективно сжимается средствами ThinLinc.
  - Включайте только в крайнем случае при очень ограниченном канале и высоком latency.

---

### 3. Настройка рабочего стола XFCE для более плавной работы

1. **Отключите визуальные эффекты и анимации**:

   - `Settings → Window Manager Tweaks → Эффекты → отключить`

2. **Упростите тему оформления**:

   - `Settings → Desktop → Цвет: сплошной цвет, чёрный; Стиль: отсутствует`

---

### 4. Дополнительное

#### Exit/Relay-узлы в Tailscale

- Можно задать **exit-node** (узел, через который будет идти весь трафик) ближе к клиенту или серверу:
  - Например, это виртуалка с Tailscale в своей стране, через которую подключение происходит на рабочий сервер, тем самым снижает общую задержку.
  - Используйте `tailscale up --exit-node=<node>` на клиенте

#### Сетевое оборудование

- Сетевое оборудование: при возможности улучшите сам канал: проверьте качество кабеля/оптики, замените Wi‑Fi на провод (Ethernet).
- Включите приоритезацию UDP (QoS) на роутере при возможности.

---

### 5. Переключение ThinLinc на NVIDIA + VirtualGL

> Цель: рендерить графику в ThinLinc на NVIDIA и зайдействовать VirtualGL.

На **сервере** (где запускается XFCE‑сессия):

* Проприетарный драйвер NVIDIA установлен, `nvidia-smi` работает.
* Установлен VirtualGL (даёт библиотеку `libvglfaker.so`).
* Актуальные версии ThinLinc (сервер/клиент).

> Если `libvglfaker.so` не находится — см. раздел «Диагностика».

#### Создание `~/.thinlinc/xstartup`

На сервере под **своим** пользователем:

```bash
mkdir -p ~/.thinlinc/
nano -w ~/.thinlinc/xstartup
```

Вставьте **полностью**:

```sh
#!/bin/sh
# ~/.thinlinc/xstartup

xhost +SI:localuser:"$USER" >/dev/null 2>&1 || true

# Enable VirtualGL
export VGL_DISPLAY=egl
export LD_PRELOAD=libvglfaker.so

# Enable NVIDIA
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __VK_LAYER_NV_optimus=NVIDIA_only

# Optimization OpenGL / VGL
export __GL_THREADED_OPTIMIZATIONS=1
export __GL_YIELD="NOTHING"
export __GL_MaxFramesAllowed=1
export __GL_SYNC_TO_VBLANK=0
export __GL_SHADER_DISK_CACHE=1

# Enable CUDA
export CUDA_DEVICE_ORDER=PCI_BUS_ID
export CUDA_VISIBLE_DEVICES=GPU-0804dd33-d461-27fe-04a1-741f23ccb013

# Starting Session
exec startxfce4 &
exec nvidia-settings --load-config-only &
wait
```

Сохраните (Ctrl+O), закройте (Ctrl+X) и сделайте файл исполняемым:

```bash
chmod +x ~/.thinlinc/xstartup
```

#### Пересоздание сессии

Сбросим сохранённую XFCE-сессию и автозапуски:

```
rm -rf ~/.cache/sessions/*
mv ~/.config/autostart ~/.config/autostart.bak 2>/dev/null || true
```

Сбросим темы GTK у пользователя к дефолтным (иначе могут вызвать чёрный экран при входе в сессию):

```
mv ~/.config/gtk-3.0 ~/.config/gtk-3.0.bak 2>/dev/null || true
mv ~/.config/gtk-4.0 ~/.config/gtk-4.0.bak 2>/dev/null || true
```

Полностью завершаем активную сессию:

```
loginctl terminate-user $USER
```

> **В клиенте ThinLinc при входе поставьте галочку, чтобы переменные окружения применились.**

#### Проверка работы

Внутри новой сессии выполните:

```bash
glxinfo | grep -i "OpenGL renderer"
```

Ожидаемо должно быть что-то вроде `OpenGL renderer string: NVIDIA ...` (а не `llvmpipe`/`software`).

Далее проверьте, что подхватился VirtualGL:

```bash
glxgears & sleep 1
pid=$(pgrep -n glxgears)
grep -F libvglfaker.so /proc/$pid/maps | head -n1
```

Строка с `libvglfaker.so` подтверждает, что графика идёт через VirtualGL. При желании посмотрите загрузку GPU:

```bash
nvidia-smi
```
```bash
nvtop
```

---

> #### Диагностика и решение возможных проблем
>
> * **`libvglfaker.so: cannot open shared object file`**  
>   Найдите точный путь к библиотеке и пропишите его в `LD_PRELOAD`:
>
>   ```bash
>   ldconfig -p | grep vglfaker
>   # /usr/lib/x86_64-linux-gnu/VirtualGL/libvglfaker.so
>   # /usr/lib64/VirtualGL/libvglfaker.so
>   ```
>
>   Затем отредактируйте `~/.thinlinc/xstartup` и укажите абсолютный путь.
>
> * **`OpenGL renderer` = llvmpipe/mesa**  
>   Проверьте:
>   - драйвер NVIDIA работает: `nvidia-smi`;
>   - переменные окружения выставлены;
>   - сессия пересоздана.
>
> #### Откат
> Закомментируйте строки в `~/.thinlinc/xstartup` и пересоздайте сессию.

---
