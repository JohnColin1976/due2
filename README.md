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

---

## Инструкция инженеру: прошивка SAM3X8E через T113

Ниже рабочий порядок обновления `due_app` на SAM3X8E через UART с платы T113.

1. Подготовить актуальный бинарник приложения для SAM3X8E:
   - собрать проект `due*`;
   - убедиться, что есть файл `build/due_app.bin`.

2. Подготовить updater на T113:
   - вариант A (рекомендуется): использовать ARM-утилиту из `ecu_gw_t113/src/tools/build/t113-static/uart_bl_update`;
   - вариант B: собрать `tools/uart_bl_update` прямо на T113 в нужном проекте `due*`.

3. Скопировать `due_app.bin` на T113 (например, в `/tmp/due_app.bin`).

4. Подключить T113 к целевому SAM3X8E по UART и определить порт Linux на T113 (обычно `/dev/ttyS4`, скорость `115200`).

5. Запустить прошивку на T113:

```bash
uart_bl_update --port /dev/ttyS4 --baud 115200 --firmware /tmp/due_app.bin
```

Если bootloader уже активен, запускать с `--no-enter-boot`:

```bash
uart_bl_update --port /dev/ttyS4 --baud 115200 --firmware /tmp/due_app.bin --no-enter-boot
```

6. Проверить успешность по логам updater:
   - `SYNC: OK`
   - `INFO: ...`
   - `ERASE: OK`
   - `WRITE: ...`
   - `VERIFY: OK`
   - `RUN: OK`

7. Подтвердить запуск приложения на диагностическом UART (FTDI, PA8/PA9, 115200 8N1):
   - `APP start`
   - `APP alive ms=...`

Замечания:
- если нужен только тест связи с bootloader без прошивки, запускайте без `--firmware`;
- для прошивки без автозапуска приложения добавьте `--no-run`;
- файл `due_app.bin` на T113 должен быть именно из актуальной сборки, иначе после `RUN` останется старое поведение.
