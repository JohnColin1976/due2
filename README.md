Проект рассчитан на сборку и отладку в WSL2 (Ubuntu 24.04+) и на Linux-хосте.

Необходимые пакеты:
- `cmake`
- `ninja-build`
- `gcc-arm-none-eabi`
- `gdb-multiarch`
- `openocd`

Установка (Ubuntu):
```bash
sudo apt update
sudo apt install -y cmake ninja-build gcc-arm-none-eabi gdb-multiarch openocd
```

Сборка:
```bash
cmake -G Ninja -S . -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake -DCMAKE_BUILD_TYPE=Release
cmake --build build --target all -j
```

Target-ы сборки:
- `due_bootloader` (`build/due_bootloader.elf`)
- `due_app` (`build/due_app.elf`)

VS Code задачи прошивки (WSL/Linux):
- `Flash Bootloader (OpenOCD+J-Link, SAM3X8E, Due2)`
- `Flash APP (OpenOCD+J-Link, SAM3X8E, Due2)`
- `Flash Full (BL+APP, OpenOCD+J-Link, Due2)`

Примечания по J-Link:
- Для `openocd` используется конфиг `tools/openocd_jlink_swd_sam3x.cfg` (J-Link + SWD + SAM3X).
- Скрипты `tools/jlink_flash_sam3x8e*.jlink` используют относительные пути (`build/...`) и не требуют правки `C:\...` путей.

Обновление через UART (`/dev/ttyS4`) с хоста:
```bash
python3 tools/uart_bl_update.py --port /dev/ttyS4 --baud 115200 --firmware build/due_app.bin
```

Обновление через UART без Python-зависимостей (C utility):
```bash
cd tools
make
./uart_bl_update --port /dev/ttyS4 --baud 115200 --firmware ../build/due_app.bin
```

Проверенный рабочий сценарий (T113 + FTDI):
1. Прошить `due_bootloader.elf` и `due_app.elf` (через task `Flash Full`).
2. На T113 запустить:
```bash
./uart_bl_update --port /dev/ttyS4 --baud 115200 --firmware due_app.bin
```
3. Ожидаемый вывод updater:
- `SYNC: OK`
- `INFO: ...`
- `ERASE: OK`
- `WRITE: ...`
- `VERIFY: OK`
- `RUN: OK`
4. В FTDI терминале (PA8/PA9, 115200 8N1) после `RUN`:
- `APP start`
- `APP alive ms=...` (примерно 1 раз/сек).

Примечание:
- Убедитесь, что на T113 используется актуальный `due_app.bin` (размер должен совпадать с файлом из `build/`), иначе после `RUN` можно видеть старое поведение APP.
