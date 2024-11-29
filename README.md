# Лабораторная работа №4: Взаимодействие с сервером

**Выполнила**: Торопчина

**Язык программирования**: Java  

## Цель работы
Изучить использование сетевых запросов и баз данных в Android-приложении. В рамках работы создано приложение, которое:
1. Проверяет подключение к интернету при запуске.
2. Отправляет запрос к API каждые 20 секунд, получает данные о текущей песне и сохраняет их в базу данных.
3. Позволяет просматривать сохраненные записи через графический интерфейс.

## Задание

### Часть 1: Проверка подключения к интернету
- При запуске приложения осуществляется проверка подключения к интернету.
- Если подключения нет, пользователю выводится уведомление о запуске в автономном режиме (Toast), где доступны только сохраненные ранее записи.

### Часть 2: Периодический запрос к серверу
- Приложение отправляет POST-запрос к API `http://media.ifmo.ru/api_get_current_song.php` каждые 20 секунд.
- Если песня отличается от последней записи в базе, она добавляется в базу данных.

### Часть 3: Отображение данных
- Приложение предоставляет интерфейс для просмотра сохраненных записей.
- Данные отображаются в виде списка, включающего исполнителя, название песни и время добавления записи.

## Технологии и инструменты
- **Android Studio**
- **Библиотеки**:
  - Retrofit для работы с API
  - Room для работы с базой данных
  - LiveData для отслеживания изменений в данных

## Структура приложения

### 1. Класс `ApiService`
Сетевой интерфейс для взаимодействия с API. Реализован с использованием библиотеки Retrofit.

**Пример метода для получения текущей песни:**
```java
@FormUrlEncoded
@POST("api_get_current_song.php")
Call<ApiResponse> getCurrentSong(
    @Field("login") String login,
    @Field("password") String password
);
```

### 2. Класс `Track`
Сущность базы данных (Entity), представляющая запись о песне.

**Пример структуры таблицы:**
```java
@Entity(tableName = "tracks")
public class Track {
    @PrimaryKey(autoGenerate = true)
    private int id;
    private String artist;
    private String title;
    private long timestamp;

    // Конструкторы, геттеры и сеттеры
}
```

### 3. Класс `TrackDao`
Интерфейс доступа к данным. Включает методы для вставки, получения всех треков и получения последнего трека.

**Пример метода для получения последней записи:**
```java
@Query("SELECT * FROM tracks ORDER BY timestamp DESC LIMIT 1")
Track getLastTrack();
```

### 4. Класс `AppDatabase`
Класс базы данных, созданный с использованием Room. Связывает `TrackDao` и `Track`.

```java
@Database(entities = {Track.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract TrackDao trackDao();
}
```

### 5. Класс `TrackRepository`
Реализует логику получения данных от API и добавления их в базу. Также предоставляет `LiveData` для обновления UI.

**Пример метода для добавления новой записи:**
```java
public void fetchAndAddCurrentSong() {
    apiService.getCurrentSong("4707login", "4707pass").enqueue(new Callback<ApiResponse>() {
        @Override
        public void onResponse(Call<ApiResponse> call, Response<ApiResponse> response) {
            if (response.isSuccessful() && response.body() != null) {
                String[] songInfo = response.body().getInfo().split(" – ");
                String artist = songInfo[0];
                String title = songInfo[1];
                long timestamp = System.currentTimeMillis();

                new Thread(() -> {
                    Track lastTrack = trackDao.getLastTrack();
                    if (lastTrack == null || !lastTrack.getTitle().equals(title)) {
                        trackDao.insertTrack(new Track(artist, title, timestamp));
                    }
                }).start();
            }
        }

        @Override
        public void onFailure(Call<ApiResponse> call, Throwable t) {
            // Логирование ошибки
        }
    });
}
```

### 6. `MainActivity`
Главное активити, отвечающее за:
- Проверку подключения к интернету.
- Запуск периодического опроса сервера.
- Отображение данных из базы с помощью `RecyclerView`.

**Пример проверки подключения к интернету:**
```java
private boolean isOnline() {
    ConnectivityManager cm = (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
    NetworkInfo activeNetwork = cm.getActiveNetworkInfo();
    return activeNetwork != null && activeNetwork.isConnectedOrConnecting();
}
```

**Пример запуска опроса сервера:**
```java
private void startFetchingSongs() {
    handler = new Handler();
    fetchTask = new Runnable() {
        @Override
        public void run() {
            trackRepository.fetchAndAddCurrentSong();
            handler.postDelayed(this, 20000); // Повтор каждые 20 секунд
        }
    };
    handler.post(fetchTask);
}
```

### 7. Макеты
#### `activity_main.xml`
Макет главного экрана с заголовком и `RecyclerView` для отображения записей:

```xml
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Список песен"
        android:textSize="24sp"
        android:layout_gravity="center_horizontal" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewTracks"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="8dp" />
</LinearLayout>
```

#### `item_track.xml`
Макет для отображения одной записи о песне:

```xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="8dp">

    <TextView
        android:id="@+id/textViewArtist"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Исполнитель"
        android:textSize="16sp" />

    <TextView
        android:id="@+id/textViewTitle"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Название песни"
        android:textSize="14sp" />
</LinearLayout>
```

## Установка и запуск

1. Склонировать репозиторий:
   ```
   bash git clone https://github.com/NoFirstReal/Lab4
   ```
2. Открыть проект в Android Studio.
3. Нажать **Run** для запуска приложения.

## Результаты
В ходе выполнения работы было создано приложение, которое:
1. Проверяет подключение к интернету.
2. Осуществляет автоматический опрос API каждые 20 секунд.
3. Сохраняет записи о песнях в базе данных.
4. Отображает список песен через удобный интерфейс.
