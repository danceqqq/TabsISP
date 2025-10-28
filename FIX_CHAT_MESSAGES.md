# Исправление: Личные сообщения в TAB меню

## Проблема

В блоке "Последние сообщения чата" у игроков отображались сообщения **от всех игроков**, а не только свои личные.

### Причина

Было **две** проблемы:

1. **Проблема 1 (исправлена ранее)**: В `ModNetworking.java`, обработчик `ChatMessagePacket` на клиенте добавлял сообщение ко **всем** `PlayerData` в `playerDataMap`:
   ```java
   // БЫЛО (неправильно):
   for (PlayerData data : playerDataMap.values()) {
       data.addChatMessageWithSender(msg.getPlayerUUID(), msg.getMessage());
   }
   
   // СТАЛО (правильно):
   PlayerData senderData = playerDataMap.get(msg.getPlayerUUID());
   if (senderData != null) {
       senderData.addChatMessageWithSender(msg.getPlayerUUID(), msg.getMessage());
   }
   ```

2. **Проблема 2 (основная)**: При логине игрока, сервер отправлял **всем** клиентам `PlayerDataSyncPacket` с **полными данными** каждого игрока, **включая их чат-сообщения**. Это приводило к тому, что чат-сообщения синхронизировались между всеми клиентами.

## Решение

### Шаг 1: Создан метод `saveForSync()` в `PlayerData.java`

Создан отдельный метод сохранения **без чат-сообщений** для синхронизации между клиентами:

```java
// Новый метод в PlayerData.java
public CompoundTag saveForSync() {
    CompoundTag nbt = new CompoundTag();
    // ... сохраняем все данные (статистика, цвета, фоны и т.д.)
    // НЕ сохраняем чат-сообщения
    return nbt;
}
```

### Шаг 2: Обновлен `PlayerDataSyncPacket`

Конструктор пакета теперь использует `saveForSync()` вместо `save()`:

```java
public PlayerDataSyncPacket(UUID playerUUID, PlayerData data) {
    this.playerUUID = playerUUID;
    // Используем saveForSync() чтобы НЕ отправлять чат-сообщения другим игрокам
    this.playerDataNBT = data.saveForSync();
}
```

### Шаг 3: Обновлен обработчик на клиенте

При получении `PlayerDataSyncPacket` на клиенте, старые чат-сообщения **сохраняются** и **восстанавливаются** после загрузки новых данных:

```java
public static void handle(PlayerDataSyncPacket msg, ...) {
    // Сохраняем старые чат-сообщения
    PlayerData existingData = playerDataMap.get(msg.playerUUID);
    List<String> oldChatMessages = null;
    if (existingData != null) {
        oldChatMessages = new ArrayList<>(existingData.getChatMessages());
    }
    
    // Загружаем новые данные (без чата)
    PlayerData data = PlayerData.load(msg.playerDataNBT);
    
    // Восстанавливаем старые чат-сообщения
    if (oldChatMessages != null) {
        for (String message : oldChatMessages) {
            data.addChatMessage(message);
        }
    }
    
    playerDataMap.put(msg.playerUUID, data);
}
```

## Результат

Теперь:
- ✅ Чат-сообщения **не синхронизируются** между клиентами через `PlayerDataSyncPacket`
- ✅ Каждый игрок видит **только свои** сообщения в блоке "Последние сообщения чата"
- ✅ Чат-сообщения **сохраняются** локально на каждом клиенте
- ✅ При обновлении других данных игрока (статистика, цвета и т.д.) чат-сообщения **не теряются**

## Измененные файлы

1. **PlayerData.java**:
   - Добавлен метод `saveForSync()` (строки 212-236)

2. **PlayerDataSyncPacket.java**:
   - Конструктор использует `saveForSync()` (строки 20-24)
   - Обработчик сохраняет и восстанавливает чат (строки 36-64)

3. **ModNetworking.java** (исправлено ранее):
   - Обработчик `ChatMessagePacket` добавляет сообщение только отправителю (строки 59-68)

## Тестирование

1. Запустите два клиента (или сервер + клиент)
2. Зайдите двумя игроками
3. Первый игрок пишет сообщение
4. Откройте TAB меню у обоих игроков
5. **Ожидаемый результат**:
   - У первого игрока в блоке "Последние сообщения чата" есть его сообщение
   - У второго игрока в блоке "Последние сообщения чата" **пусто** (или только его сообщения)

## Дополнительная информация

### Почему не удаляем чат полностью из `save()`?

Метод `save()` используется для **сохранения на диск** (в `PersistentData`), поэтому чат-сообщения должны сохраняться, чтобы игрок видел свои старые сообщения после перезахода.

### Что произойдет при перезаходе игрока?

При перезаходе:
1. Сервер загружает `PlayerData` из `PersistentData` (с чатом)
2. Сервер отправляет `PlayerDataSyncPacket` всем клиентам (без чата)
3. Локальный клиент игрока загружает свои данные из `PersistentData` (с чатом)

Таким образом, каждый игрок сохраняет свои личные сообщения между сессиями.

