# Часть 1. Сервисы и их жизненный цикл (Темы 1–2)

Везде пакет `com.university.app`. Чтобы не повторяться, общие вещи:

- Зависимость для `ServiceCompat` / `NotificationCompat`: `androidx.core:core:1.13.1` (или новее).
- Каждый сервис нужно объявить в `AndroidManifest.xml` (показано у каждого задания).
- Activity с интерфейсом собраны кодом, без XML-разметки, чтобы файл был самодостаточным.

---

## Тема 1

### 1.1 Служба таймера обратного отсчёта (Started Service)

Сервис запускает поток, который считает 60 → 0 и пишет в Logcat. Поток нужен, потому что сервис по умолчанию работает в главном потоке, и `Thread.sleep` там вызвал бы ANR. В конце `stopSelf(startId)`.

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

public class CountdownService extends Service {
    private static final String TAG = "CountdownService";
    private volatile boolean running = false;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (running) {
            return START_NOT_STICKY; // уже считаем, второй поток не нужен
        }
        running = true;
        new Thread(() -> {
            for (int i = 60; i >= 0 && running; i--) {
                Log.d(TAG, "Осталось секунд: " + i);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    break;
                }
            }
            stopSelf(startId);
        }, "countdown-thread").start();
        return START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        running = false;
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

Манифест:
```xml
<service android:name=".services.CountdownService" android:exported="false" />
```
Запуск (из видимой Activity): `startService(new Intent(this, CountdownService.class));`

---

### 1.2 Foreground-сервис аудиоплеера (`mediaPlayback`)

Манифест:
```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service
    android:name=".services.MusicForegroundService"
    android:exported="false"
    android:foregroundServiceType="mediaPlayback" />
```

```java
package com.university.app.services;

import android.app.Notification;
import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.os.IBinder;

import androidx.core.app.NotificationChannelCompat;
import androidx.core.app.NotificationCompat;
import androidx.core.app.NotificationManagerCompat;
import androidx.core.app.ServiceCompat;

public class MusicForegroundService extends Service {
    public static final String CHANNEL_ID = "music_channel";
    public static final String EXTRA_TITLE = "title";
    private static final int NOTIFICATION_ID = 201;

    @Override
    public void onCreate() {
        super.onCreate();
        NotificationChannelCompat channel = new NotificationChannelCompat.Builder(
                CHANNEL_ID, NotificationManagerCompat.IMPORTANCE_LOW)
                .setName("Музыкальный плеер")
                .build();
        NotificationManagerCompat.from(this).createNotificationChannel(channel);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String title = (intent != null) ? intent.getStringExtra(EXTRA_TITLE) : null;
        if (title == null) {
            title = "Неизвестная мелодия";
        }

        Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                .setContentTitle("Сейчас играет")
                .setContentText(title)
                .setSmallIcon(android.R.drawable.ic_media_play)
                .setOngoing(true)
                .build();

        // Вызвать нужно в течение 5 секунд после startForegroundService()
        ServiceCompat.startForeground(this, NOTIFICATION_ID, notification,
                ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK);

        // Здесь запускается MediaPlayer
        return START_NOT_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Запуск:
```java
Intent i = new Intent(this, MusicForegroundService.class);
i.putExtra(MusicForegroundService.EXTRA_TITLE, "Bohemian Rhapsody");
androidx.core.content.ContextCompat.startForegroundService(this, i);
```

---

### 1.3 Безопасная остановка через `stopSelf(startId)`

Если во время скачивания пришёл второй запрос, `stopSelf()` без номера убил бы сервис вместе с ним. `stopSelf(startId)` останавливает сервис, только если этот `startId` — последний выданный. Очередь задач одна (`SingleThreadExecutor`), поэтому запросы идут по очереди.

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

import java.io.IOException;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class DownloadService extends Service {
    private static final String TAG = "DownloadService";
    public static final String EXTRA_URL = "url";
    private ExecutorService executor;

    @Override
    public void onCreate() {
        super.onCreate();
        executor = Executors.newSingleThreadExecutor();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        final String link = (intent != null) ? intent.getStringExtra(EXTRA_URL) : null;
        if (link == null) {
            stopSelf(startId);
            return START_NOT_STICKY;
        }
        executor.execute(() -> {
            download(link);
            stopSelf(startId); // остановится только если новых запросов не было
            Log.d(TAG, "stopSelf(" + startId + ")");
        });
        return START_REDELIVER_INTENT;
    }

    private void download(String link) {
        HttpURLConnection conn = null;
        try {
            conn = (HttpURLConnection) new URL(link).openConnection();
            conn.setConnectTimeout(10_000);
            conn.setReadTimeout(10_000);
            long total = 0;
            try (InputStream in = conn.getInputStream()) {
                byte[] buf = new byte[8192];
                int n;
                while ((n = in.read(buf)) != -1) {
                    total += n; // байты читаем и считаем, на диск не пишем
                }
            }
            Log.d(TAG, "Передано байт: " + total);
        } catch (IOException e) {
            Log.e(TAG, "Ошибка загрузки", e);
        } finally {
            if (conn != null) {
                conn.disconnect();
            }
        }
    }

    @Override
    public void onDestroy() {
        executor.shutdownNow();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Манифест: `<service android:name=".services.DownloadService" android:exported="false" />` и `<uses-permission android:name="android.permission.INTERNET" />`.

---

### 1.4 Сервис в отдельном процессе `:downloader`

```xml
<service
    android:name=".services.RemoteDownloaderService"
    android:exported="false"
    android:process=":downloader" />
```

```java
package com.university.app.services;

import android.app.Application;
import android.app.Service;
import android.content.Intent;
import android.os.Build;
import android.os.IBinder;
import android.os.Process;
import android.util.Log;

public class RemoteDownloaderService extends Service {
    private static final String TAG = "RemoteDownloader";

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String processName = (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P)
                ? Application.getProcessName() : "n/a";
        Log.d(TAG, "PID = " + Process.myPid() + ", процесс = " + processName);
        stopSelf(startId);
        return START_NOT_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
В Logcat PID сервиса будет отличаться от PID Activity (выведите `Process.myPid()` в `MainActivity.onCreate()` и сравните). Важный побочный эффект: `Application.onCreate()` вызывается **отдельно для каждого процесса**, то есть статические поля в процессах `:downloader` и основном — разные объекты.

---

### 1.5 Проверка `POST_NOTIFICATIONS` перед `startForegroundService()`

Начиная с Android 13 разрешение надо запрашивать в рантайме. Если пользователь откажет, сам сервис всё равно запустится, но уведомление не появится в шторке (оно будет только в диспетчере задач). Поэтому запрашиваем разрешение до старта.

```java
package com.university.app;

import android.Manifest;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.Bundle;
import android.widget.Button;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

import com.university.app.services.LocationTrackerService;

public class MainActivity extends AppCompatActivity {

    private final ActivityResultLauncher<String> notificationPermission =
            registerForActivityResult(new ActivityResultContracts.RequestPermission(),
                    granted -> startTracker()); // стартуем в любом случае

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button = new Button(this);
        button.setText("Запустить трекер");
        button.setOnClickListener(v -> startTrackerSafely());
        setContentView(button);
    }

    private void startTrackerSafely() {
        boolean needRequest = Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU
                && ContextCompat.checkSelfPermission(this, Manifest.permission.POST_NOTIFICATIONS)
                != PackageManager.PERMISSION_GRANTED;
        if (needRequest) {
            notificationPermission.launch(Manifest.permission.POST_NOTIFICATIONS);
        } else {
            startTracker();
        }
    }

    private void startTracker() {
        ContextCompat.startForegroundService(this,
                new Intent(this, LocationTrackerService.class));
    }
}
```
`LocationTrackerService` — класс из примера в вашем файле.

---

## Тема 2

### 2.1 Bound-сервис со случайным числом

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;

import java.util.Random;

public class RandomService extends Service {
    private final IBinder binder = new LocalBinder();
    private final Random random = new Random();

    public class LocalBinder extends Binder {
        public RandomService getService() {
            return RandomService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    public int getRandomNumber() {
        return random.nextInt(100);
    }
}
```

```java
package com.university.app;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.services.RandomService;

public class RandomActivity extends AppCompatActivity {
    private RandomService service;
    private boolean bound = false;
    private TextView textView;

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder binder) {
            service = ((RandomService.LocalBinder) binder).getService();
            bound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            bound = false;
            service = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LinearLayout root = new LinearLayout(this);
        root.setOrientation(LinearLayout.VERTICAL);
        root.setPadding(32, 32, 32, 32);

        textView = new TextView(this);
        textView.setTextSize(28);
        textView.setText("—");

        Button button = new Button(this);
        button.setText("Получить число");
        button.setOnClickListener(v -> {
            if (bound) {
                textView.setText(String.valueOf(service.getRandomNumber()));
            }
        });

        root.addView(textView);
        root.addView(button);
        setContentView(root);
    }

    @Override
    protected void onStart() {
        super.onStart();
        bindService(new Intent(this, RandomService.class), connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (bound) {
            unbindService(connection);
            bound = false;
        }
    }
}
```
Манифест: `<service android:name=".services.RandomService" android:exported="false" />`

---

### 2.2 Эксперимент с `START_STICKY`

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.util.Log;

public class StickyCounterService extends Service {
    private static final String TAG = "StickyCounter";
    private final Handler handler = new Handler(Looper.getMainLooper());
    private int seconds = 0;

    private final Runnable tick = new Runnable() {
        @Override
        public void run() {
            Log.d(TAG, "Секунд: " + (++seconds));
            handler.postDelayed(this, 1000);
        }
    };

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent == null) {
            Log.w(TAG, "ПЕРЕЗАПУСК системой: intent == null, счётчик начат заново");
        } else {
            Log.d(TAG, "Обычный запуск, startId=" + startId);
        }
        handler.removeCallbacks(tick);
        handler.post(tick);
        return START_STICKY;
    }

    @Override
    public void onDestroy() {
        handler.removeCallbacks(tick);
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Проверка:
1. Запустите сервис и сверните приложение.
2. `adb shell am kill com.university.app` (или `adb shell kill <PID>` — PID смотрите в `adb shell pidof com.university.app`).
3. Через несколько секунд в Logcat появится строка «ПЕРЕЗАПУСК системой: intent == null», и счётчик начнётся с 1.

Важно: `adb shell am force-stop` и кнопка «Остановить» в Android Studio перезапуск **блокируют**, потому что это «принудительная остановка пользователем», а не нехватка памяти.

---

### 2.3 `onUnbind()` → `true` и `onRebind()`

`onRebind()` вызовется только если сервис **не был уничтожен** после отвязки. Поэтому сервис должен быть ещё и запущенным (`startService`), иначе после отвязки последнего клиента он умрёт.

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.util.Log;

public class RebindService extends Service {
    private static final String TAG = "RebindService";
    private final IBinder binder = new Binder();

    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind (первое подключение)");
        return binder;
    }

    @Override
    public boolean onUnbind(Intent intent) {
        Log.d(TAG, "onUnbind, все клиенты отвязались");
        return true; // просим систему вызвать onRebind() при новом подключении
    }

    @Override
    public void onRebind(Intent intent) {
        super.onRebind(intent);
        Log.d(TAG, "onRebind (клиент подключился снова)");
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "onDestroy");
        super.onDestroy();
    }
}
```

```java
package com.university.app;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.services.RebindService;

public class RebindActivity extends AppCompatActivity {
    private boolean bound = false;

    private final ServiceConnection connection = new ServiceConnection() {
        @Override public void onServiceConnected(ComponentName n, IBinder b) { bound = true; }
        @Override public void onServiceDisconnected(ComponentName n) { bound = false; }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // Сервис должен быть запущенным, иначе он умрёт после unbind
        startService(new Intent(this, RebindService.class));
    }

    @Override
    protected void onStart() {
        super.onStart();
        bindService(new Intent(this, RebindService.class), connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (bound) {
            unbindService(connection);
            bound = false;
        }
    }
}
```
Как проверить: откройте экран, сверните (в логе `onUnbind`), вернитесь (в логе `onRebind`, а не `onBind`). Чтобы завершить сервис, вызовите `stopService(...)`.

---

### 2.4 Счётчик одновременно подключённых клиентов

Тонкость: `onBind()` вызывается **только для первого клиента** (для одного и того же `Intent` система кэширует `IBinder`), а `onUnbind()` — только когда отвязался **последний**. Поэтому считать клиентов в этих методах нельзя. Считаем через собственные методы `attach()`/`detach()` в Binder.

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.util.Log;

import java.util.concurrent.atomic.AtomicInteger;

public class ClientCounterService extends Service {
    private static final String TAG = "ClientCounter";
    private final AtomicInteger clients = new AtomicInteger(0);
    private final IBinder binder = new CounterBinder();

    public class CounterBinder extends Binder {
        public void attach() {
            Log.d(TAG, "Клиент подключился. Всего: " + clients.incrementAndGet());
        }

        public void detach() {
            Log.d(TAG, "Клиент отключился. Всего: " + clients.decrementAndGet());
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

```java
package com.university.app;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.IBinder;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.services.ClientCounterService;

// Базовый экран: от него наследуются ScreenA и ScreenB
public abstract class CounterClientActivity extends AppCompatActivity {
    private ClientCounterService.CounterBinder binder;
    private boolean bound = false;

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName n, IBinder service) {
            binder = (ClientCounterService.CounterBinder) service;
            binder.attach();
            bound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName n) {
            bound = false;
            binder = null;
        }
    };

    @Override
    protected void onStart() {
        super.onStart();
        bindService(new Intent(this, ClientCounterService.class), connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (bound) {
            binder.detach();
            unbindService(connection);
            bound = false;
        }
    }
}

// Два экрана-клиента (каждый в своём файле, оба объявить в манифесте)
// public class ScreenA extends CounterClientActivity { }
// public class ScreenB extends CounterClientActivity { }
```
Откройте ScreenA, из неё запустите ScreenB: в логе будет «Всего: 2», при возврате — «Всего: 1».

---

### 2.5 Комбинированный сервис (Started + Bound)

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.util.Log;

public class StatusService extends Service {
    private static final String TAG = "StatusService";
    private final IBinder binder = new StatusBinder();
    private final long startedAt = System.currentTimeMillis();

    public class StatusBinder extends Binder {
        public StatusService getService() {
            return StatusService.this;
        }
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d(TAG, "Запущен через startService — переживёт закрытие экрана");
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    public String getStatus() {
        long seconds = (System.currentTimeMillis() - startedAt) / 1000;
        return "Работает уже " + seconds + " сек.";
    }
}
```

```java
package com.university.app;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.services.StatusService;

public class StatusActivity extends AppCompatActivity {
    private StatusService service;
    private boolean bound = false;
    private TextView statusView;

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName n, IBinder b) {
            service = ((StatusService.StatusBinder) b).getService();
            bound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName n) {
            bound = false;
            service = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LinearLayout root = new LinearLayout(this);
        root.setOrientation(LinearLayout.VERTICAL);
        root.setPadding(32, 32, 32, 32);

        statusView = new TextView(this);
        Button read = new Button(this);
        read.setText("Прочитать статус");
        read.setOnClickListener(v -> {
            if (bound) statusView.setText(service.getStatus());
        });
        Button stop = new Button(this);
        stop.setText("Остановить сервис");
        stop.setOnClickListener(v -> stopService(new Intent(this, StatusService.class)));

        root.addView(statusView);
        root.addView(read);
        root.addView(stop);
        setContentView(root);

        // 1) Сначала запускаем, чтобы сервис не умер при выходе с экрана
        startService(new Intent(this, StatusService.class));
    }

    @Override
    protected void onStart() {
        super.onStart();
        // 2) Временно привязываемся, чтобы читать статус
        bindService(new Intent(this, StatusService.class), connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (bound) {
            unbindService(connection);
            bound = false;
        }
    }
}
```
Сервис остановится только когда вы нажмёте «Остановить» (`stopService`) **и** не останется привязанных клиентов.


---

# Часть 2. Broadcast Receivers и их жизненный цикл (Темы 3–4)

Общее правило для динамических приёмников: регистрируем в `onStart()`, снимаем в `onStop()`. Для совместимости со всеми версиями используем `ContextCompat.registerReceiver(...)`, он сам подставляет флаг экспорта на старых API. Для системных событий (батарея, режим полёта) безопаснее `RECEIVER_NOT_EXPORTED`: системные рассылки он не блокирует, а чужим приложениям не даёт слать вам поддельные интенты.

---

## Тема 3

### 3.1 Приёмник `ACTION_BATTERY_LOW`

`ACTION_BATTERY_LOW` нельзя ловить из манифеста, только динамически.

```java
package com.university.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;
import android.widget.TextView;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

public class BatteryActivity extends AppCompatActivity {
    private TextView status;

    private final BroadcastReceiver batteryLowReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (Intent.ACTION_BATTERY_LOW.equals(intent.getAction())) {
                status.setText("⚠ Низкий заряд батареи!");
                Toast.makeText(context, "Батарея разряжена, подключите зарядку",
                        Toast.LENGTH_LONG).show();
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        status = new TextView(this);
        status.setTextSize(22);
        status.setPadding(32, 32, 32, 32);
        status.setText("Ожидаем событие батареи...");
        setContentView(status);
    }

    @Override
    protected void onStart() {
        super.onStart();
        ContextCompat.registerReceiver(this, batteryLowReceiver,
                new IntentFilter(Intent.ACTION_BATTERY_LOW),
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    protected void onStop() {
        super.onStop();
        unregisterReceiver(batteryLowReceiver);
    }
}
```
Проверка на эмуляторе: Extended controls → Battery → уровень ниже 15 %.

---

### 3.2 Режим полёта (`ACTION_AIRPLANE_MODE_CHANGED`)

```java
package com.university.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

public class AirplaneActivity extends AppCompatActivity {
    private TextView view;

    private final BroadcastReceiver receiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (Intent.ACTION_AIRPLANE_MODE_CHANGED.equals(intent.getAction())) {
                boolean on = intent.getBooleanExtra("state", false); // true = включён
                view.setText(on ? "Режим полёта: ВКЛ" : "Режим полёта: ВЫКЛ");
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        view = new TextView(this);
        view.setTextSize(22);
        view.setPadding(32, 32, 32, 32);
        view.setText("Переключите режим полёта");
        setContentView(view);
    }

    @Override
    protected void onStart() {
        super.onStart();
        ContextCompat.registerReceiver(this, receiver,
                new IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED),
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    protected void onStop() {
        super.onStop();
        unregisterReceiver(receiver);
    }
}
```
Ключ `"state"` — это документированный extra этого бродкаста. Текущее состояние без события: `Settings.Global.getInt(getContentResolver(), Settings.Global.AIRPLANE_MODE_ON, 0) == 1`.

---

### 3.3 Статический приёмник перезагрузки

Манифест:
```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

<receiver
    android:name=".receivers.BootReceiver"
    android:enabled="true"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```
`exported="true"` здесь нужно, потому что событие приходит от системы. Это безопасно, пока вы проверяете `getAction()`.

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class BootReceiver extends BroadcastReceiver {
    private static final String TAG = "BootReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        if (Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            Log.d(TAG, "Устройство загрузилось (после разблокировки пользователем)");
        }
    }
}
```
Проверка: `adb shell am broadcast -a android.intent.action.BOOT_COMPLETED -p com.university.app`

---

### 3.4 Кастомный внутрипрограммный бродкаст

С Android 8 неявные бродкасты в манифест-приёмники не доставляются. Поэтому интент делаем «почти явным» через `setPackage()` и регистрируем оба приёмника динамически.

```java
package com.university.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;
import android.util.Log;
import android.widget.Button;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

public class CustomBroadcastActivity extends AppCompatActivity {
    public static final String ACTION_ALERT = "com.university.CUSTOM_ALERT";
    private static final String TAG = "CustomBroadcast";

    private final BroadcastReceiver receiverOne = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            Log.d(TAG, "Приёмник №1: " + intent.getStringExtra("msg"));
        }
    };

    private final BroadcastReceiver receiverTwo = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            Log.d(TAG, "Приёмник №2: " + intent.getStringExtra("msg"));
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button send = new Button(this);
        send.setText("Отправить CUSTOM_ALERT");
        send.setOnClickListener(v -> {
            Intent intent = new Intent(ACTION_ALERT);
            intent.setPackage(getPackageName()); // только внутри приложения
            intent.putExtra("msg", "Внимание! Это тестовое оповещение");
            sendBroadcast(intent);
        });
        setContentView(send);
    }

    @Override
    protected void onStart() {
        super.onStart();
        IntentFilter filter = new IntentFilter(ACTION_ALERT);
        ContextCompat.registerReceiver(this, receiverOne, filter, ContextCompat.RECEIVER_NOT_EXPORTED);
        ContextCompat.registerReceiver(this, receiverTwo, filter, ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    protected void onStop() {
        super.onStop();
        unregisterReceiver(receiverOne);
        unregisterReceiver(receiverTwo);
    }
}
```

---

### 3.5 Упорядоченный бродкаст и `abortBroadcast()`

Приоритет задаётся в манифесте. Чем больше число, тем раньше получит. Здесь приёмник с приоритетом 200 срабатывает первым и обрывает цепочку, поэтому приёмник с 100 ничего не получит.

Манифест:
```xml
<receiver android:name=".receivers.HighPriorityReceiver" android:exported="false">
    <intent-filter android:priority="200">
        <action android:name="com.university.ORDERED_ALERT" />
    </intent-filter>
</receiver>

<receiver android:name=".receivers.LowPriorityReceiver" android:exported="false">
    <intent-filter android:priority="100">
        <action android:name="com.university.ORDERED_ALERT" />
    </intent-filter>
</receiver>
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class HighPriorityReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.d("Ordered", "Приоритет 200: получил первым, обрываю цепочку");
        abortBroadcast(); // уберите эту строку, и сработает приёмник с приоритетом 100
    }
}
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class LowPriorityReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.d("Ordered", "Приоритет 100: получил сообщение");
    }
}
```

Отправка (из Activity):
```java
Intent intent = new Intent("com.university.ORDERED_ALERT");
intent.setPackage(getPackageName());
sendOrderedBroadcast(intent, null);
```
Если `exported="false"`, а интент отправлен с `setPackage(getPackageName())`, доставка работает (отправитель — ваше же приложение).

---

## Тема 4

### 4.1 Проверка, что `onReceive()` на главном потоке

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.os.Looper;
import android.util.Log;

public class ThreadCheckReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        boolean onMain = Looper.myLooper() == Looper.getMainLooper();
        Log.d("ThreadCheck", "Главный поток? " + onMain
                + " | поток: " + Thread.currentThread().getName());
        // Ожидаемый вывод: true | main
    }
}
```
Регистрируйте любым способом из заданий выше (например, на `ACTION_AIRPLANE_MODE_CHANGED`). Вывод всегда `true`, поэтому тяжёлую работу в `onReceive()` делать нельзя.

---

### 4.2 Имитация ANR

Только для учебного проекта.

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class AnrReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.d("AnrReceiver", "Старт блокировки главного потока...");
        try {
            Thread.sleep(15_000); // больше лимита в 10 секунд
        } catch (InterruptedException ignored) {
        }
        Log.d("AnrReceiver", "Сюда мы можем и не дойти — система убьёт процесс раньше");
    }
}
```
Что смотреть: после ~10 секунд появится диалог «Приложение не отвечает» и в Logcat строка вида `ANR in com.university.app` с причиной `Broadcast of Intent { ... }`. Полный отчёт лежит в `/data/anr/` (на эмуляторе: `adb pull /data/anr/`). Если приёмник вызван в фоне, лимит бывает больше (до 60 с), поэтому для чистого эксперимента отправляйте его из активного приложения.

---

### 4.3 Передача тяжёлой работы в WorkManager

Зависимость: `androidx.work:work-runtime:2.9.1`.

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;

import androidx.work.OneTimeWorkRequest;
import androidx.work.WorkManager;

import com.university.app.work.HeavyWorker;

public class WorkEnqueueReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        // Ничего тяжёлого здесь: только ставим задачу в очередь
        OneTimeWorkRequest request = new OneTimeWorkRequest.Builder(HeavyWorker.class).build();
        WorkManager.getInstance(context).enqueue(request);
    }
}
```

```java
package com.university.app.work;

import android.content.Context;
import android.util.Log;

import androidx.annotation.NonNull;
import androidx.work.Worker;
import androidx.work.WorkerParameters;

public class HeavyWorker extends Worker {
    public HeavyWorker(@NonNull Context context, @NonNull WorkerParameters params) {
        super(context, params);
    }

    @NonNull
    @Override
    public Result doWork() {
        // doWork() выполняется в фоновом потоке WorkManager
        try {
            Thread.sleep(5000); // имитация тяжёлой операции
        } catch (InterruptedException e) {
            return Result.retry();
        }
        Log.d("HeavyWorker", "Тяжёлая работа завершена");
        return Result.success();
    }
}
```
Почему так: WorkManager сам держит процесс живым и умеет повторять задачу, а голый `new Thread()` в приёмнике после выхода из `onReceive()` система может убить вместе с процессом.

---

### 4.4 Асинхронная запись в SharedPreferences через `goAsync()`

Манифест:
```xml
<receiver android:name=".receivers.BootTimeReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```
(и `RECEIVE_BOOT_COMPLETED` в `uses-permission`)

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;

public class BootTimeReceiver extends BroadcastReceiver {
    public static final String PREFS = "boot_prefs";
    public static final String KEY_BOOT_TIME = "last_boot_time";

    @Override
    public void onReceive(Context context, Intent intent) {
        if (!Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            return;
        }
        final PendingResult pending = goAsync();
        final Context appContext = context.getApplicationContext();

        new Thread(() -> {
            try {
                SharedPreferences prefs =
                        appContext.getSharedPreferences(PREFS, Context.MODE_PRIVATE);
                // commit() синхронный: запись точно завершится до finish()
                prefs.edit().putLong(KEY_BOOT_TIME, System.currentTimeMillis()).commit();
            } finally {
                pending.finish(); // обязательно, иначе приёмник "повиснет" до таймаута
            }
        }).start();
    }
}
```
Здесь `new Thread()` допустим только потому, что мы взяли `goAsync()`: он сообщает системе «я ещё работаю». Работа должна укладываться в секунды.

---

### 4.5 Утечка при отсутствии `unregisterReceiver()`

```java
package com.university.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;
import android.util.Log;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

public class LeakDemoActivity extends AppCompatActivity {

    // Анонимный/внутренний класс держит неявную ссылку на LeakDemoActivity
    private final BroadcastReceiver receiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            Log.d("LeakDemo", "Получено в " + LeakDemoActivity.this);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ContextCompat.registerReceiver(this, receiver,
                new IntentFilter(Intent.ACTION_BATTERY_LOW),
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    // ОШИБКА: onDestroy()/onStop() без unregisterReceiver(receiver)
    // Правильно: вызвать unregisterReceiver(receiver) в onStop() (если регистрировали в onStart())
}
```
Что происходит: система хранит приёмник в списке зарегистрированных, а приёмник через `LeakDemoActivity.this` держит Activity. Когда Activity закрыли, GC не может её собрать, и вместе с ней висит вся иерархия View.

Как увидеть на практике:
1. Запустите, затем закройте экран кнопкой «Назад».
2. В Logcat появится ошибка `Activity com.university.app.LeakDemoActivity has leaked IntentReceiver ... Are you missing a call to unregisterReceiver()?`
3. Для надёжной проверки подключите LeakCanary (`debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.14'`): после закрытия экрана он покажет цепочку ссылок до Activity.


---

# Часть 3. Content Provider, UriMatcher, жизненный цикл (Темы 5–6)

Задания 5.3, 5.4, 6.2, 6.3 и 6.5 — это разные куски **одного** провайдера, поэтому ниже сначала идёт полный рабочий `NotesProvider` (на SQLite) с пометками `// ЗАДАНИЕ 5.3` и т.д., а потом отдельные клиенты и пояснения. Так проще сдавать и запускать.

---

## Общий класс: NotesProvider (SQLite + UriMatcher)

Манифест:
```xml
<provider
    android:name=".providers.NotesProvider"
    android:authorities="com.university.app.provider.notes"
    android:exported="true"
    android:readPermission="com.university.permission.READ_NOTES"
    android:writePermission="com.university.permission.WRITE_NOTES" />

<permission android:name="com.university.permission.READ_NOTES"
    android:protectionLevel="signature" />
<permission android:name="com.university.permission.WRITE_NOTES"
    android:protectionLevel="signature" />
<uses-permission android:name="com.university.permission.READ_NOTES" />
<uses-permission android:name="com.university.permission.WRITE_NOTES" />
```
Свои же `uses-permission` нужны, чтобы Activity этого же приложения проходила проверку. Если провайдер нужен только внутри приложения, поставьте `android:exported="false"` и уберите разрешения.

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentProviderOperation;
import android.content.ContentProviderResult;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.Context;
import android.content.OperationApplicationException;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.DatabaseUtils;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.net.Uri;
import android.os.SystemClock;
import android.util.Log;

import java.util.ArrayList;

public class NotesProvider extends ContentProvider {

    public static final String AUTHORITY = "com.university.app.provider.notes";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/notes");

    private static final String TABLE = "notes";
    private static final int CODE_NOTES_DIR = 1;
    private static final int CODE_NOTE_ITEM = 2;
    private static final int CODE_CATEGORY = 3;

    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        MATCHER.addURI(AUTHORITY, "notes", CODE_NOTES_DIR);
        MATCHER.addURI(AUTHORITY, "notes/#", CODE_NOTE_ITEM);
        // ЗАДАНИЕ 5.3: маска * — любая строка (категория)
        MATCHER.addURI(AUTHORITY, "notes/category/*", CODE_CATEGORY);
    }

    private static class Db extends SQLiteOpenHelper {
        Db(Context context) {
            super(context, "notes.db", null, 1);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE notes ("
                    + "_id INTEGER PRIMARY KEY AUTOINCREMENT,"
                    + "title TEXT NOT NULL,"
                    + "body TEXT,"
                    + "category TEXT NOT NULL DEFAULT 'general')");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int oldV, int newV) {
            db.execSQL("DROP TABLE IF EXISTS notes");
            onCreate(db);
        }
    }

    private Db helper;
    private final Object writeLock = new Object();

    @Override
    public boolean onCreate() {
        // ЗАДАНИЕ 6.1: лог для сравнения порядка инициализации
        Log.d("InitOrder", "NotesProvider.onCreate  t=" + SystemClock.uptimeMillis());
        // Конструктор SQLiteOpenHelper лёгкий: файл БД открывается лениво
        // при первом getReadableDatabase()/getWritableDatabase().
        helper = new Db(getContext());
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        SQLiteDatabase db = helper.getReadableDatabase();
        Cursor cursor;
        switch (MATCHER.match(uri)) {
            case CODE_NOTES_DIR:
                cursor = db.query(TABLE, projection, selection, selectionArgs, null, null, sortOrder);
                break;
            case CODE_NOTE_ITEM:
                cursor = db.query(TABLE, projection, whereWithId(uri, selection),
                        selectionArgs, null, null, sortOrder);
                break;
            case CODE_CATEGORY:
                // ЗАДАНИЕ 6.5: значение из URI передаётся через ?, а не склеивается
                cursor = db.query(TABLE, projection,
                        DatabaseUtils.concatenateWhere("category = ?", selection),
                        DatabaseUtils.appendSelectionArgs(
                                new String[]{uri.getLastPathSegment()}, selectionArgs),
                        null, null, sortOrder);
                break;
            default:
                throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        cursor.setNotificationUri(getContext().getContentResolver(), uri);
        return cursor;
    }

    @Override
    public String getType(Uri uri) {
        switch (MATCHER.match(uri)) {
            case CODE_NOTES_DIR:
            case CODE_CATEGORY:
                return "vnd.android.cursor.dir/vnd.com.university.note";
            case CODE_NOTE_ITEM:
                return "vnd.android.cursor.item/vnd.com.university.note";
            default:
                throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        if (MATCHER.match(uri) != CODE_NOTES_DIR) {
            throw new IllegalArgumentException("Вставка не поддерживается для URI: " + uri);
        }
        // ЗАДАНИЕ 5.4: проверка обязательного поля
        String title = (values == null) ? null : values.getAsString("title");
        if (title == null || title.trim().isEmpty()) {
            throw new IllegalArgumentException("Поле 'title' обязательно");
        }
        long id;
        // ЗАДАНИЕ 6.2: короткая критическая секция
        synchronized (writeLock) {
            id = helper.getWritableDatabase().insertOrThrow(TABLE, null, values);
        }
        getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return ContentUris.withAppendedId(CONTENT_URI, id);
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        String where;
        switch (MATCHER.match(uri)) {
            case CODE_NOTES_DIR:
                where = selection;
                break;
            case CODE_NOTE_ITEM:
                where = whereWithId(uri, selection);
                break;
            default:
                throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        int rows;
        synchronized (writeLock) {
            rows = helper.getWritableDatabase().update(TABLE, values, where, selectionArgs);
        }
        if (rows > 0) {
            getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        }
        return rows;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        String where;
        switch (MATCHER.match(uri)) {
            case CODE_NOTES_DIR:
                where = selection;
                break;
            case CODE_NOTE_ITEM:
                where = whereWithId(uri, selection);
                break;
            default:
                throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        int rows;
        synchronized (writeLock) {
            rows = helper.getWritableDatabase().delete(TABLE, where, selectionArgs);
        }
        if (rows > 0) {
            getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        }
        return rows;
    }

    // ЗАДАНИЕ 6.3: пакетная вставка в одной транзакции
    @Override
    public ContentProviderResult[] applyBatch(ArrayList<ContentProviderOperation> operations)
            throws OperationApplicationException {
        SQLiteDatabase db = helper.getWritableDatabase();
        db.beginTransaction();
        try {
            ContentProviderResult[] results = super.applyBatch(operations);
            db.setTransactionSuccessful(); // без этого вызова всё откатится
            return results;
        } finally {
            db.endTransaction();
        }
    }

    // id — число (long), поэтому склейка тут безопасна
    private static String whereWithId(Uri uri, String selection) {
        return DatabaseUtils.concatenateWhere("_id = " + ContentUris.parseId(uri), selection);
    }
}
```

---

## Тема 5

### 5.1 Чтение системных контактов

Манифест: `<uses-permission android:name="android.permission.READ_CONTACTS" />`

```java
package com.university.app;

import android.Manifest;
import android.content.pm.PackageManager;
import android.database.Cursor;
import android.os.Bundle;
import android.provider.ContactsContract;
import android.util.Log;
import android.widget.Button;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

public class ContactsActivity extends AppCompatActivity {

    private final ActivityResultLauncher<String> permission =
            registerForActivityResult(new ActivityResultContracts.RequestPermission(), granted -> {
                if (granted) {
                    readContacts();
                } else {
                    Log.w("Contacts", "Доступ к контактам не выдан");
                }
            });

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button = new Button(this);
        button.setText("Показать контакты в логе");
        button.setOnClickListener(v -> {
            if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_CONTACTS)
                    == PackageManager.PERMISSION_GRANTED) {
                readContacts();
            } else {
                permission.launch(Manifest.permission.READ_CONTACTS);
            }
        });
        setContentView(button);
    }

    private void readContacts() {
        String[] projection = {
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME,
                ContactsContract.CommonDataKinds.Phone.NUMBER
        };
        try (Cursor cursor = getContentResolver().query(
                ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
                projection, null, null,
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME + " ASC")) {
            if (cursor == null) {
                return;
            }
            int nameIdx = cursor.getColumnIndexOrThrow(
                    ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME);
            int numIdx = cursor.getColumnIndexOrThrow(
                    ContactsContract.CommonDataKinds.Phone.NUMBER);
            while (cursor.moveToNext()) {
                Log.d("Contacts", cursor.getString(nameIdx) + " — " + cursor.getString(numIdx));
            }
        }
    }
}
```
Для больших списков вынесите `readContacts()` в фоновый поток.

---

### 5.2 Медиатека (MediaStore): названия и размеры фото

Манифест:
```xml
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
```

```java
package com.university.app;

import android.Manifest;
import android.content.pm.PackageManager;
import android.database.Cursor;
import android.os.Build;
import android.os.Bundle;
import android.provider.MediaStore;
import android.util.Log;
import android.widget.Button;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

public class MediaActivity extends AppCompatActivity {

    private final ActivityResultLauncher<String> permission =
            registerForActivityResult(new ActivityResultContracts.RequestPermission(), granted -> {
                if (granted) {
                    readImages();
                }
            });

    private String requiredPermission() {
        return Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU
                ? Manifest.permission.READ_MEDIA_IMAGES
                : Manifest.permission.READ_EXTERNAL_STORAGE;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button = new Button(this);
        button.setText("Показать фото");
        button.setOnClickListener(v -> {
            if (ContextCompat.checkSelfPermission(this, requiredPermission())
                    == PackageManager.PERMISSION_GRANTED) {
                readImages();
            } else {
                permission.launch(requiredPermission());
            }
        });
        setContentView(button);
    }

    private void readImages() {
        String[] projection = {MediaStore.Images.Media.DISPLAY_NAME, MediaStore.Images.Media.SIZE};
        try (Cursor c = getContentResolver().query(
                MediaStore.Images.Media.EXTERNAL_CONTENT_URI, projection, null, null,
                MediaStore.Images.Media.DATE_ADDED + " DESC")) {
            if (c == null) {
                return;
            }
            int nameIdx = c.getColumnIndexOrThrow(MediaStore.Images.Media.DISPLAY_NAME);
            int sizeIdx = c.getColumnIndexOrThrow(MediaStore.Images.Media.SIZE);
            int count = 0;
            while (c.moveToNext() && count++ < 50) { // первые 50 строк
                Log.d("Media", c.getString(nameIdx) + " — " + (c.getLong(sizeIdx) / 1024) + " КБ");
            }
        }
    }
}
```
Не добавляйте `LIMIT 50` в `sortOrder`: на Android 11+ такой приём бросает исключение. Ограничивайте количество в цикле или через `Bundle` с `QUERY_ARG_LIMIT`.

---

### 5.3 `UriMatcher` с маской `*` (`notes/category/*`)

Реализовано в `NotesProvider` выше: строка `MATCHER.addURI(AUTHORITY, "notes/category/*", CODE_CATEGORY)` и ветка `case CODE_CATEGORY`. Отличие масок: `#` — только число, `*` — любая строка.

Проверка клиентом:
```java
Uri uri = Uri.withAppendedPath(NotesProvider.CONTENT_URI, "category/work");
try (Cursor c = getContentResolver().query(uri, null, null, null, "_id DESC")) {
    while (c != null && c.moveToNext()) {
        Log.d("Notes", c.getString(c.getColumnIndexOrThrow("title")));
    }
}
```

---

### 5.4 Валидация в `insert()`

Реализовано в `NotesProvider.insert()`: если `title` пустой или отсутствует, бросается `IllegalArgumentException`. Исключение дойдёт до клиента, поэтому клиент должен его ловить:

```java
ContentValues values = new ContentValues();
values.put("body", "Заметка без заголовка");
try {
    getContentResolver().insert(NotesProvider.CONTENT_URI, values);
} catch (IllegalArgumentException e) {
    Log.e("Notes", "Отклонено провайдером: " + e.getMessage());
}
```

---

### 5.5 `ContentObserver` в Activity

```java
package com.university.app;

import android.database.ContentObserver;
import android.database.Cursor;
import android.net.Uri;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.providers.NotesProvider;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class NotesObserverActivity extends AppCompatActivity {
    private TextView list;
    private final ExecutorService executor = Executors.newSingleThreadExecutor();

    private final ContentObserver observer = new ContentObserver(new Handler(Looper.getMainLooper())) {
        @Override
        public void onChange(boolean selfChange, Uri uri) {
            reload(); // провайдер вызвал notifyChange()
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        list = new TextView(this);
        list.setPadding(32, 32, 32, 32);
        setContentView(list);
    }

    @Override
    protected void onStart() {
        super.onStart();
        getContentResolver().registerContentObserver(NotesProvider.CONTENT_URI, true, observer);
        reload();
    }

    @Override
    protected void onStop() {
        super.onStop();
        getContentResolver().unregisterContentObserver(observer);
    }

    @Override
    protected void onDestroy() {
        executor.shutdownNow();
        super.onDestroy();
    }

    private void reload() {
        executor.execute(() -> {
            StringBuilder sb = new StringBuilder();
            try (Cursor c = getContentResolver().query(NotesProvider.CONTENT_URI,
                    new String[]{"_id", "title"}, null, null, "_id DESC")) {
                while (c != null && c.moveToNext()) {
                    sb.append(c.getLong(0)).append(". ").append(c.getString(1)).append('\n');
                }
            }
            runOnUiThread(() -> list.setText(sb.length() == 0 ? "Заметок нет" : sb.toString()));
        });
    }
}
```
Чтобы увидеть работу, вставьте заметку из другого места (кнопка, `adb shell content insert --uri content://com.university.app.provider.notes/notes --bind title:s:Test`): экран обновится сам.

---

## Тема 6

### 6.1 Порядок инициализации и временные метки

```java
package com.university.app;

import android.app.Application;
import android.content.Context;
import android.os.SystemClock;
import android.util.Log;

public class App extends Application {
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        Log.d("InitOrder", "App.attachBaseContext t=" + SystemClock.uptimeMillis());
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d("InitOrder", "App.onCreate           t=" + SystemClock.uptimeMillis());
    }
}
```
В манифесте: `<application android:name=".App" ...>`. В `MainActivity.onCreate()` добавьте:
```java
Log.d("InitOrder", "MainActivity.onCreate  t=" + SystemClock.uptimeMillis());
```
(`NotesProvider.onCreate()` уже содержит лог.) Фильтр Logcat: `tag:InitOrder`. Ожидаемый порядок при холодном старте:

```
App.attachBaseContext
NotesProvider.onCreate     <- раньше Application.onCreate()!
App.onCreate
MainActivity.onCreate
```
Вывод: в `ContentProvider.onCreate()` нельзя рассчитывать на то, что `Application.onCreate()` уже выполнился (например, на инициализированные там синглтоны).

---

### 6.2 Синхронизация CRUD и риски для пула Binder

Реализовано в `NotesProvider` (`synchronized (writeLock)` вокруг записи). Пояснение простыми словами:

- Запросы от других приложений приходят в пул потоков Binder (по умолчанию 16 потоков на процесс).
- Если один поток держит `synchronized` долго (например, считает что-то тяжёлое внутри блока), остальные потоки встают в очередь на блокировку. Когда заняты все 16, новые обращения к провайдеру (и к другим Binder-сервисам вашего процесса) ждут. Клиентское приложение зависает на `query()` и может получить ANR.
- Худший случай — взаимная блокировка (deadlock), если блокировки берутся в разном порядке.
- Поэтому: блокировка только вокруг самой записи, никакой сети и тяжёлых вычислений внутри, один общий замок, а не несколько.
- Практическая заметка: `SQLiteDatabase` сам сериализует записи, так что `synchronized` в учебном примере нужен для демонстрации и для защиты вашего собственного общего состояния (кэши, счётчики), а не для самой БД.

---

### 6.3 Пакетная вставка с транзакцией

Реализовано в `NotesProvider.applyBatch()`. Клиентский вызов:

```java
package com.university.app;

import android.content.ContentProviderOperation;
import android.content.OperationApplicationException;
import android.os.RemoteException;
import android.util.Log;

import com.university.app.providers.NotesProvider;

import java.util.ArrayList;

public class BatchInsertDemo {
    public static void run(android.content.Context context) {
        ArrayList<ContentProviderOperation> ops = new ArrayList<>();
        for (int i = 1; i <= 100; i++) {
            ops.add(ContentProviderOperation.newInsert(NotesProvider.CONTENT_URI)
                    .withValue("title", "Заметка " + i)
                    .withValue("category", i % 2 == 0 ? "work" : "home")
                    .build());
        }
        try {
            context.getContentResolver().applyBatch(NotesProvider.AUTHORITY, ops);
            Log.d("Batch", "Вставлено " + ops.size() + " строк атомарно");
        } catch (RemoteException | OperationApplicationException e) {
            Log.e("Batch", "Пакет откатился целиком", e);
        }
    }
}
```
Смысл транзакции: либо вставятся все 100 строк, либо ни одной (если одна операция упала, `setTransactionSuccessful()` не вызывается и `endTransaction()` откатывает всё). Бонус: запись в одной транзакции намного быстрее отдельных вставок.

---

### 6.4 `FileProvider` для безопасной выдачи PDF

Файл `res/xml/file_paths.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <!-- Открываем только подпапку files/pdf/, а не всё хранилище -->
    <files-path name="pdf_reports" path="pdf/" />
</paths>
```

Манифест (внутри `<application>`):
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```
`exported="false"` + `grantUriPermissions="true"` — правильная пара: провайдер недоступен напрямую, но вы выдаёте временный доступ к конкретному URI.

```java
package com.university.app;

import android.content.ActivityNotFoundException;
import android.content.Intent;
import android.graphics.Paint;
import android.graphics.pdf.PdfDocument;
import android.net.Uri;
import android.os.Bundle;
import android.widget.Button;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;

public class PdfShareActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button = new Button(this);
        button.setText("Открыть PDF");
        button.setOnClickListener(v -> openPdf());
        setContentView(button);
    }

    private void openPdf() {
        File dir = new File(getFilesDir(), "pdf");
        File pdf = new File(dir, "report.pdf");
        try {
            if (!pdf.exists()) {
                createSamplePdf(dir, pdf);
            }
        } catch (IOException e) {
            Toast.makeText(this, "Не удалось создать PDF", Toast.LENGTH_SHORT).show();
            return;
        }

        Uri uri = FileProvider.getUriForFile(this, getPackageName() + ".fileprovider", pdf);
        // получится content://com.university.app.fileprovider/pdf_reports/report.pdf

        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setDataAndType(uri, "application/pdf");
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION); // временный доступ на чтение
        try {
            startActivity(intent);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, "Нет приложения для PDF", Toast.LENGTH_SHORT).show();
        }
    }

    // Только чтобы было что открывать
    private void createSamplePdf(File dir, File target) throws IOException {
        if (!dir.exists() && !dir.mkdirs()) {
            throw new IOException("Нет каталога");
        }
        PdfDocument doc = new PdfDocument();
        PdfDocument.Page page = doc.startPage(new PdfDocument.PageInfo.Builder(300, 400, 1).create());
        Paint paint = new Paint();
        paint.setTextSize(18f);
        page.getCanvas().drawText("Отчёт (пример)", 40, 60, paint);
        doc.finishPage(page);
        try (FileOutputStream out = new FileOutputStream(target)) {
            doc.writeTo(out);
        } finally {
            doc.close();
        }
    }
}
```
Если в `getUriForFile` получите `IllegalArgumentException: Failed to find configured root`, значит файл лежит вне папок, описанных в `file_paths.xml` (проверьте тег и `path`).

---

### 6.5 Защита от SQL-инъекций через `selectionArgs`

В `NotesProvider.query()` пользовательское значение попадает в `selectionArgs`, а не в строку запроса. Демонстрация «плохо / хорошо» на клиенте:

```java
package com.university.app;

import android.content.ContentResolver;
import android.database.Cursor;
import android.util.Log;

import com.university.app.providers.NotesProvider;

public class SqlInjectionDemo {

    // ПЛОХО: ввод пользователя склеивается в SQL
    public static int searchUnsafe(ContentResolver resolver, String userInput) {
        String selection = "title = '" + userInput + "'";
        // Ввод  x' OR '1'='1  превращает условие в title = 'x' OR '1'='1' — вернутся ВСЕ строки
        try (Cursor c = resolver.query(NotesProvider.CONTENT_URI, null, selection, null, null)) {
            return c == null ? 0 : c.getCount();
        }
    }

    // ХОРОШО: ? + selectionArgs, ввод всегда трактуется как данные, а не как код
    public static int searchSafe(ContentResolver resolver, String userInput) {
        try (Cursor c = resolver.query(NotesProvider.CONTENT_URI, null,
                "title = ?", new String[]{userInput}, null)) {
            return c == null ? 0 : c.getCount();
        }
    }

    public static void demo(ContentResolver resolver) {
        String attack = "x' OR '1'='1";
        Log.d("Sql", "Небезопасно найдено: " + searchUnsafe(resolver, attack));
        Log.d("Sql", "Безопасно найдено:   " + searchSafe(resolver, attack)); // 0
    }
}
```
Что ещё усиливает защиту на стороне провайдера: `SQLiteQueryBuilder` с `setStrict(true)` и `setProjectionMap(...)` (запрещает подсовывать произвольные колонки и подзапросы в `selection`/`projection`), а также белый список допустимых колонок для сортировки. Внутри самого провайдера (`whereWithId`) склеивается только `long`, полученный через `ContentUris.parseId()`, поэтому там инъекция невозможна.


---

# Часть 4. Смешанные задания 1–25 (Services, Broadcasts, Providers)

Это ядро каждого решения: рабочий компонент + манифест. Экраны минимальные. Общий помощник для уведомлений (создайте один раз, на него ссылаются многие сервисы):

```java
package com.university.app.util;

import android.app.Notification;
import android.content.Context;

import androidx.core.app.NotificationChannelCompat;
import androidx.core.app.NotificationCompat;
import androidx.core.app.NotificationManagerCompat;

public final class Notifs {
    public static final String CHANNEL = "system_components_channel";

    private Notifs() { }

    public static void ensureChannel(Context c) {
        NotificationManagerCompat.from(c).createNotificationChannel(
                new NotificationChannelCompat.Builder(CHANNEL, NotificationManagerCompat.IMPORTANCE_LOW)
                        .setName("Системные компоненты").build());
    }

    public static Notification build(Context c, String title, String text) {
        ensureChannel(c);
        return new NotificationCompat.Builder(c, CHANNEL)
                .setContentTitle(title)
                .setContentText(text)
                .setSmallIcon(android.R.drawable.stat_notify_sync)
                .setOngoing(true)
                .build();
    }
}
```

Базовые разрешения для всех foreground-сервисов: `FOREGROUND_SERVICE`, `POST_NOTIFICATIONS` + разрешение конкретного типа (указано у каждого).

---

## Блок 1. Службы

### 1. Фоновый шагомер (`TYPE_STEP_COUNTER`)

Датчик отдаёт число шагов **с момента загрузки телефона**, поэтому запоминаем стартовое значение и вычитаем. Раз в 10 секунд обновляем уведомление.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_HEALTH" />
<uses-permission android:name="android.permission.ACTIVITY_RECOGNITION" />
<service android:name=".services.StepTrackerService" android:exported="false"
    android:foregroundServiceType="health" />
```
`ACTIVITY_RECOGNITION` нужно запросить в рантайме до запуска сервиса.

```java
package com.university.app.services;

import android.app.NotificationManager;
import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.hardware.Sensor;
import android.hardware.SensorEvent;
import android.hardware.SensorEventListener;
import android.hardware.SensorManager;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;

import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

public class StepTrackerService extends Service implements SensorEventListener {
    private static final int NOTIF_ID = 301;
    private static final long UPDATE_MS = 10_000;

    private SensorManager sensorManager;
    private final Handler handler = new Handler(Looper.getMainLooper());
    private float baseline = -1f;
    private volatile int steps = 0;

    private final Runnable updater = new Runnable() {
        @Override
        public void run() {
            getSystemService(NotificationManager.class)
                    .notify(NOTIF_ID, Notifs.build(StepTrackerService.this, "Шагомер", "Шагов: " + steps));
            handler.postDelayed(this, UPDATE_MS);
        }
    };

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, NOTIF_ID,
                Notifs.build(this, "Шагомер", "Шагов: 0"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_HEALTH);

        sensorManager = getSystemService(SensorManager.class);
        Sensor stepSensor = sensorManager.getDefaultSensor(Sensor.TYPE_STEP_COUNTER);
        if (stepSensor == null) { // на устройстве нет датчика
            stopSelf();
            return START_NOT_STICKY;
        }
        sensorManager.registerListener(this, stepSensor, SensorManager.SENSOR_DELAY_NORMAL);
        handler.removeCallbacks(updater);
        handler.postDelayed(updater, UPDATE_MS);
        return START_STICKY;
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        float total = event.values[0];
        if (baseline < 0) {
            baseline = total;
        }
        steps = (int) (total - baseline);
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) { }

    @Override
    public void onDestroy() {
        handler.removeCallbacks(updater);
        if (sensorManager != null) {
            sensorManager.unregisterListener(this);
        }
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

---

### 2. Аудиоплеер с кнопками в шторке (Bound + Foreground)

Положите `track1.mp3`, `track2.mp3`, `track3.mp3` в `res/raw`. Кнопки уведомления шлют `PendingIntent` в этот же сервис, а Activity общается с ним через Binder.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<service android:name=".services.PlayerService" android:exported="false"
    android:foregroundServiceType="mediaPlayback" />
```

```java
package com.university.app.services;

import android.app.Notification;
import android.app.NotificationManager;
import android.app.PendingIntent;
import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.media.MediaPlayer;
import android.os.Binder;
import android.os.IBinder;

import androidx.core.app.NotificationCompat;
import androidx.core.app.ServiceCompat;

import com.university.app.R;
import com.university.app.util.Notifs;

public class PlayerService extends Service {
    public static final String ACTION_PLAY = "com.university.app.PLAY";
    public static final String ACTION_PAUSE = "com.university.app.PAUSE";
    public static final String ACTION_NEXT = "com.university.app.NEXT";
    private static final int NOTIF_ID = 302;
    private static final int[] TRACKS = {R.raw.track1, R.raw.track2, R.raw.track3};
    private static final String[] TITLES = {"Трек 1", "Трек 2", "Трек 3"};

    private final IBinder binder = new LocalBinder();
    private MediaPlayer player;
    private int index = 0;
    private boolean foreground = false;

    public class LocalBinder extends Binder {
        public PlayerService getService() {
            return PlayerService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        showNotification(); // сразу, чтобы уложиться в 5 секунд
        String action = (intent == null) ? null : intent.getAction();
        if (ACTION_PAUSE.equals(action)) {
            pause();
        } else if (ACTION_NEXT.equals(action)) {
            next();
        } else {
            play();
        }
        return START_NOT_STICKY;
    }

    public void play() {
        if (player == null) {
            player = MediaPlayer.create(this, TRACKS[index]);
            if (player == null) { // файл не найден
                stopSelf();
                return;
            }
            player.setOnCompletionListener(mp -> next());
        }
        player.start();
        showNotification();
    }

    public void pause() {
        if (player != null && player.isPlaying()) {
            player.pause();
        }
        showNotification();
    }

    public void next() {
        releasePlayer();
        index = (index + 1) % TRACKS.length;
        play();
    }

    public String getTitle() {
        return TITLES[index];
    }

    public boolean isPlaying() {
        return player != null && player.isPlaying();
    }

    private void showNotification() {
        Notification n = new NotificationCompat.Builder(this, Notifs.CHANNEL)
                .setContentTitle(TITLES[index])
                .setContentText(isPlaying() ? "Играет" : "Пауза")
                .setSmallIcon(android.R.drawable.ic_media_play)
                .setOngoing(true)
                .addAction(action("Play", ACTION_PLAY, android.R.drawable.ic_media_play))
                .addAction(action("Pause", ACTION_PAUSE, android.R.drawable.ic_media_pause))
                .addAction(action("Next", ACTION_NEXT, android.R.drawable.ic_media_next))
                .build();
        if (!foreground) {
            Notifs.ensureChannel(this);
            ServiceCompat.startForeground(this, NOTIF_ID, n,
                    ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK);
            foreground = true;
        } else {
            getSystemService(NotificationManager.class).notify(NOTIF_ID, n);
        }
    }

    private NotificationCompat.Action action(String title, String act, int icon) {
        Intent i = new Intent(this, PlayerService.class).setAction(act);
        PendingIntent pi = PendingIntent.getService(this, act.hashCode(), i,
                PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_UPDATE_CURRENT);
        return new NotificationCompat.Action(icon, title, pi);
    }

    private void releasePlayer() {
        if (player != null) {
            player.release();
            player = null;
        }
    }

    @Override
    public void onDestroy() {
        releasePlayer();
        super.onDestroy();
    }
}
```
Запуск из Activity: сначала `ContextCompat.startForegroundService(this, new Intent(this, PlayerService.class).setAction(PlayerService.ACTION_PLAY))`, затем `bindService(...)`, чтобы показывать название и состояние через `getService().getTitle()`.

---

### 3. Пакетная выгрузка 100 файлов, `START_REDELIVER_INTENT`, прогресс в %

Прогресс сохраняется в `SharedPreferences`, так что после убийства процесса система повторно доставит тот же `Intent` и скачивание продолжится с последнего файла.

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<service android:name=".services.PhotoBatchService" android:exported="false"
    android:foregroundServiceType="dataSync" />
```

```java
package com.university.app.services;

import android.app.Notification;
import android.app.NotificationManager;
import android.app.Service;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.pm.ServiceInfo;
import android.os.IBinder;
import android.util.Log;

import androidx.core.app.NotificationCompat;
import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

import java.io.IOException;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.Locale;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class PhotoBatchService extends Service {
    public static final String EXTRA_URL_TEMPLATE = "url_template"; // "https://example.com/photos/%d.jpg"
    private static final int TOTAL = 100;
    private static final int NOTIF_ID = 303;
    private static final String PREFS = "photo_batch";
    private static final String KEY_DONE = "done";

    private ExecutorService executor;

    @Override
    public void onCreate() {
        super.onCreate();
        executor = Executors.newSingleThreadExecutor();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        final String template = (intent == null) ? null : intent.getStringExtra(EXTRA_URL_TEMPLATE);
        if (template == null) {
            stopSelf(startId);
            return START_NOT_STICKY;
        }
        ServiceCompat.startForeground(this, NOTIF_ID, progress(0),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC);

        executor.execute(() -> {
            SharedPreferences prefs = getSharedPreferences(PREFS, Context.MODE_PRIVATE);
            int start = prefs.getInt(KEY_DONE, 0); // продолжаем после перезапуска
            for (int i = start; i < TOTAL && !Thread.currentThread().isInterrupted(); i++) {
                downloadOne(String.format(Locale.US, template, i + 1));
                prefs.edit().putInt(KEY_DONE, i + 1).apply();
                int percent = (i + 1) * 100 / TOTAL;
                getSystemService(NotificationManager.class).notify(NOTIF_ID, progress(percent));
            }
            prefs.edit().remove(KEY_DONE).apply();
            ServiceCompat.stopForeground(this, ServiceCompat.STOP_FOREGROUND_REMOVE);
            stopSelf(startId);
        });
        return START_REDELIVER_INTENT;
    }

    private Notification progress(int percent) {
        Notifs.ensureChannel(this);
        return new NotificationCompat.Builder(this, Notifs.CHANNEL)
                .setContentTitle("Загрузка фото")
                .setContentText(percent + "%")
                .setSmallIcon(android.R.drawable.stat_sys_download)
                .setOngoing(true)
                .setProgress(100, percent, false)
                .build();
    }

    private void downloadOne(String link) {
        HttpURLConnection conn = null;
        try {
            conn = (HttpURLConnection) new URL(link).openConnection();
            conn.setConnectTimeout(10_000);
            conn.setReadTimeout(10_000);
            try (InputStream in = conn.getInputStream()) {
                byte[] buf = new byte[8192];
                while (in.read(buf) != -1) {
                    // данные читаем; сохранение в нужное место — по вашему заданию
                }
            }
        } catch (IOException e) {
            Log.w("PhotoBatch", "Не удалось: " + link, e);
        } finally {
            if (conn != null) {
                conn.disconnect();
            }
        }
    }

    @Override
    public void onDestroy() {
        executor.shutdownNow();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Замечание про Android 15: у типа `dataSync` лимит около 6 часов в сутки; при превышении система вызывает `onTimeout()`, а сервис нужно остановить. Для 100 файлов это неактуально.

---

### 4. WebSocket-клиент с авто-переподключением

Зависимость: `implementation 'com.squareup.okhttp3:okhttp:4.12.0'`.

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<service android:name=".services.QuotesSocketService" android:exported="false"
    android:foregroundServiceType="dataSync" />
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.util.Log;

import androidx.annotation.NonNull;
import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

import java.util.concurrent.TimeUnit;

import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.WebSocket;
import okhttp3.WebSocketListener;

public class QuotesSocketService extends Service {
    private static final String TAG = "QuotesSocket";
    private static final String URL = "wss://example.com/quotes"; // замените на свой сервер

    private final Handler handler = new Handler(Looper.getMainLooper());
    private OkHttpClient client;
    private WebSocket socket;
    private int attempt = 0;
    private boolean stopped = false;

    @Override
    public void onCreate() {
        super.onCreate();
        client = new OkHttpClient.Builder()
                .pingInterval(20, TimeUnit.SECONDS) // быстро замечаем «тихий» обрыв
                .build();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 304,
                Notifs.build(this, "Котировки", "Подключено к бирже"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC);
        stopped = false;
        connect();
        return START_STICKY;
    }

    private void connect() {
        if (stopped) {
            return;
        }
        socket = client.newWebSocket(new Request.Builder().url(URL).build(), new WebSocketListener() {
            @Override
            public void onOpen(@NonNull WebSocket ws, @NonNull Response response) {
                attempt = 0; // успешное соединение — сбрасываем задержку
                ws.send("{\"action\":\"subscribe\",\"symbols\":[\"AAPL\",\"MSFT\"]}");
            }

            @Override
            public void onMessage(@NonNull WebSocket ws, @NonNull String text) {
                Log.d(TAG, "Котировка: " + text);
            }

            @Override
            public void onClosed(@NonNull WebSocket ws, int code, @NonNull String reason) {
                scheduleReconnect(ws);
            }

            @Override
            public void onFailure(@NonNull WebSocket ws, @NonNull Throwable t, Response r) {
                Log.w(TAG, "Обрыв: " + t.getMessage());
                scheduleReconnect(ws);
            }
        });
    }

    private void scheduleReconnect(WebSocket from) {
        handler.post(() -> {
            if (stopped || from != socket) { // игнорируем события устаревших сокетов
                return;
            }
            long delay = Math.min(30_000L, 1000L << Math.min(attempt++, 5)); // 1,2,4,8,16,30 c
            Log.d(TAG, "Переподключение через " + delay + " мс");
            handler.postDelayed(this::connect, delay);
        });
    }

    @Override
    public void onDestroy() {
        stopped = true;
        handler.removeCallbacksAndMessages(null);
        if (socket != null) {
            socket.close(1000, "service stopped");
        }
        client.dispatcher().executorService().shutdown();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Честное замечание: тип `dataSync` не рассчитан на соединение «навсегда» (лимит на Android 15). Для круглосуточного соединения на практике берут `specialUse` (с обоснованием в Play Console) или переходят на push (FCM).

---

### 5. Сервис геозоны: сигнал при выходе из радиуса 500 м

Используем встроенный `LocationManager` (без Google Play Services). Срабатывает именно на **переход** «внутри → снаружи», а не на каждое обновление.

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<service android:name=".services.GeofenceMonitorService" android:exported="false"
    android:foregroundServiceType="location" />
```
Разрешение `ACCESS_FINE_LOCATION` запросите в Activity **до** запуска сервиса.

```java
package com.university.app.services;

import android.Manifest;
import android.app.Service;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.content.pm.ServiceInfo;
import android.location.Location;
import android.location.LocationListener;
import android.location.LocationManager;
import android.media.AudioManager;
import android.media.ToneGenerator;
import android.os.Bundle;
import android.os.IBinder;
import android.util.Log;

import androidx.annotation.NonNull;
import androidx.core.app.ServiceCompat;
import androidx.core.content.ContextCompat;

import com.university.app.util.Notifs;

public class GeofenceMonitorService extends Service implements LocationListener {
    private static final double WAREHOUSE_LAT = 54.1930; // координаты склада — замените
    private static final double WAREHOUSE_LON = 37.6177;
    private static final float RADIUS_M = 500f;

    private LocationManager locationManager;
    private ToneGenerator tone;
    private boolean wasInside = true;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 305,
                Notifs.build(this, "Контроль зоны склада", "Радиус 500 м"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION);

        if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION)
                != PackageManager.PERMISSION_GRANTED) {
            stopSelf();
            return START_NOT_STICKY;
        }
        tone = new ToneGenerator(AudioManager.STREAM_ALARM, 100);
        locationManager = getSystemService(LocationManager.class);
        try {
            locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 5000L, 10f, this);
        } catch (SecurityException | IllegalArgumentException e) {
            Log.e("Geofence", "GPS недоступен", e);
            stopSelf();
        }
        return START_STICKY;
    }

    @Override
    public void onLocationChanged(@NonNull Location location) {
        float[] result = new float[1];
        Location.distanceBetween(location.getLatitude(), location.getLongitude(),
                WAREHOUSE_LAT, WAREHOUSE_LON, result);
        boolean inside = result[0] <= RADIUS_M;
        if (wasInside && !inside) {
            Log.w("Geofence", "ВЫХОД из зоны! Расстояние: " + (int) result[0] + " м");
            tone.startTone(ToneGenerator.TONE_CDMA_ALERT_CALL_GUARD, 1500);
        }
        wasInside = inside;
    }

    // Пустые методы нужны для совместимости со старыми версиями Android
    @Override public void onProviderEnabled(@NonNull String provider) { }
    @Override public void onProviderDisabled(@NonNull String provider) { }
    @Override public void onStatusChanged(String provider, int status, Bundle extras) { }

    @Override
    public void onDestroy() {
        if (locationManager != null) {
            locationManager.removeUpdates(this);
        }
        if (tone != null) {
            tone.release();
        }
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

---

### 6. Очередь перекодирования видео (`newSingleThreadExecutor`)

Очередь и жизненный цикл сервиса готовы полностью. Само «сжатие» в учебном варианте имитируется; в боевом приложении на его место ставят `androidx.media3:media3-transformer` (запуск `Transformer.start(...)`).

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<service android:name=".services.VideoCompressService" android:exported="false"
    android:foregroundServiceType="dataSync" />
```

```java
package com.university.app.services;

import android.app.NotificationManager;
import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.os.IBinder;
import android.util.Log;

import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

public class VideoCompressService extends Service {
    public static final String EXTRA_PATH = "video_path";
    private static final int NOTIF_ID = 306;

    private ExecutorService queue;
    private final AtomicInteger pending = new AtomicInteger(0);

    @Override
    public void onCreate() {
        super.onCreate();
        queue = Executors.newSingleThreadExecutor(); // задачи идут строго по одной
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        final String path = (intent == null) ? null : intent.getStringExtra(EXTRA_PATH);
        int inQueue = pending.incrementAndGet();
        ServiceCompat.startForeground(this, NOTIF_ID,
                Notifs.build(this, "Сжатие видео", "В очереди: " + inQueue),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC);

        queue.execute(() -> {
            try {
                if (path != null) {
                    compress(path);
                }
            } finally {
                int left = pending.decrementAndGet();
                getSystemService(NotificationManager.class).notify(NOTIF_ID,
                        Notifs.build(this, "Сжатие видео", "В очереди: " + left));
                stopSelf(startId); // остановится, только когда очередь пуста (startId последний)
            }
        });
        return START_NOT_STICKY;
    }

    private void compress(String path) {
        Log.d("VideoCompress", "Старт: " + path);
        try {
            Thread.sleep(3000); // здесь вызов настоящего транскодера
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        Log.d("VideoCompress", "Готово: " + path);
    }

    @Override
    public void onDestroy() {
        queue.shutdownNow();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

---

### 7. IPC через `Messenger` (клиент в другом процессе)

```xml
<service android:name=".services.MessengerService" android:exported="false"
    android:process=":remote" />
```
`exported="false"`: клиент в **том же приложении**, но другом процессе. Для сторонних приложений нужен `exported="true"` и `android:permission`.

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Bundle;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.os.Message;
import android.os.Messenger;
import android.os.Process;
import android.os.RemoteException;
import android.util.Log;

public class MessengerService extends Service {
    public static final int MSG_SAY_HELLO = 1;
    public static final int MSG_REPLY = 2;

    private static class IncomingHandler extends Handler {
        IncomingHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            if (msg.what == MSG_SAY_HELLO) {
                String name = msg.getData().getString("name", "аноним");
                Message reply = Message.obtain(null, MSG_REPLY);
                Bundle data = new Bundle();
                data.putString("text", "Привет, " + name + "! Отвечает процесс " + Process.myPid());
                reply.setData(data);
                try {
                    if (msg.replyTo != null) {
                        msg.replyTo.send(reply);
                    }
                } catch (RemoteException e) {
                    Log.w("MessengerService", "Клиент пропал", e);
                }
            } else {
                super.handleMessage(msg);
            }
        }
    }

    private final Messenger messenger = new Messenger(new IncomingHandler(Looper.getMainLooper()));

    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

Клиент:
```java
package com.university.app;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.os.Message;
import android.os.Messenger;
import android.os.RemoteException;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.services.MessengerService;

public class MessengerActivity extends AppCompatActivity {
    private Messenger server;
    private boolean bound = false;
    private TextView output;

    // Сюда придёт ответ сервиса
    private final Messenger clientMessenger = new Messenger(new Handler(Looper.getMainLooper(), msg -> {
        if (msg.what == MessengerService.MSG_REPLY) {
            output.setText(msg.getData().getString("text"));
            return true;
        }
        return false;
    }));

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder binder) {
            server = new Messenger(binder);
            bound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            server = null;
            bound = false;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LinearLayout root = new LinearLayout(this);
        root.setOrientation(LinearLayout.VERTICAL);
        root.setPadding(32, 32, 32, 32);
        output = new TextView(this);
        Button send = new Button(this);
        send.setText("Отправить сообщение");
        send.setOnClickListener(v -> {
            if (!bound) return;
            Message msg = Message.obtain(null, MessengerService.MSG_SAY_HELLO);
            Bundle data = new Bundle();
            data.putString("name", "Артём");
            msg.setData(data);
            msg.replyTo = clientMessenger;
            try {
                server.send(msg);
            } catch (RemoteException e) {
                output.setText("Сервис недоступен");
            }
        });
        root.addView(output);
        root.addView(send);
        setContentView(root);
    }

    @Override
    protected void onStart() {
        super.onStart();
        bindService(new Intent(this, MessengerService.class), connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (bound) {
            unbindService(connection);
            bound = false;
        }
    }
}
```
Через `Messenger` все сообщения обрабатываются **по одному** в потоке `Handler` (для параллельности нужен AIDL).

---

### 8. Очистка кэша: файлы старше 7 дней

```xml
<service android:name=".services.CacheCleanerService" android:exported="false" />
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

import java.io.File;

public class CacheCleanerService extends Service {
    private static final String TAG = "CacheCleaner";
    private static final long MAX_AGE_MS = 7L * 24 * 60 * 60 * 1000;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        new Thread(() -> {
            long cutoff = System.currentTimeMillis() - MAX_AGE_MS;
            for (File dir : new File[]{getCacheDir(), getExternalCacheDir()}) {
                if (dir == null) continue;
                long before = dirSize(dir);
                int deleted = deleteOld(dir, cutoff);
                Log.d(TAG, dir + ": было " + before / 1024 + " КБ, удалено файлов: " + deleted
                        + ", стало " + dirSize(dir) / 1024 + " КБ");
            }
            stopSelf(startId);
        }, "cache-cleaner").start();
        return START_NOT_STICKY;
    }

    private long dirSize(File dir) {
        long size = 0;
        File[] files = dir.listFiles();
        if (files == null) return 0;
        for (File f : files) {
            size += f.isDirectory() ? dirSize(f) : f.length();
        }
        return size;
    }

    private int deleteOld(File dir, long cutoff) {
        int count = 0;
        File[] files = dir.listFiles();
        if (files == null) return 0;
        for (File f : files) {
            if (f.isDirectory()) {
                count += deleteOld(f, cutoff);
                String[] rest = f.list();
                if (rest != null && rest.length == 0) {
                    f.delete(); // убираем опустевшие папки
                }
            } else if (f.lastModified() < cutoff && f.delete()) {
                count++;
            }
        }
        return count;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Запускать нужно из видимой Activity (при старте приложения). Для регулярной очистки «по расписанию» надёжнее `PeriodicWorkRequest` в WorkManager.

---

### 9. Мониторинг буфера обмена: телефонный номер → быстрый звонок

**Важное ограничение:** начиная с Android 10 приложение в фоне **не может читать** буфер обмена (читать можно, только пока приложение в фокусе или оно является клавиатурой по умолчанию). Поэтому фоновая «слежка» на современных телефонах не работает по задумке системы. Код ниже корректно работает, пока приложение на переднем плане.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<service android:name=".services.ClipboardWatchService" android:exported="false"
    android:foregroundServiceType="specialUse">
    <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="clipboard_phone_helper" />
</service>
```

```java
package com.university.app.services;

import android.app.NotificationManager;
import android.app.PendingIntent;
import android.app.Service;
import android.content.ClipData;
import android.content.ClipboardManager;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.net.Uri;
import android.os.IBinder;

import androidx.core.app.NotificationCompat;
import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class ClipboardWatchService extends Service {
    private static final Pattern PHONE = Pattern.compile("^\\+?[0-9][0-9\\s\\-()]{6,18}[0-9]$");
    private ClipboardManager clipboard;

    private final ClipboardManager.OnPrimaryClipChangedListener listener = () -> {
        ClipData clip = clipboard.getPrimaryClip(); // в фоне на Android 10+ вернёт null
        if (clip == null || clip.getItemCount() == 0) return;
        CharSequence text = clip.getItemAt(0).coerceToText(this);
        if (text == null) return;
        String candidate = text.toString().trim();
        Matcher m = PHONE.matcher(candidate);
        if (m.matches()) {
            offerCall(candidate);
        }
    };

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 309,
                Notifs.build(this, "Помощник буфера обмена", "Слежу за номерами"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE);
        if (clipboard == null) {
            clipboard = getSystemService(ClipboardManager.class);
            clipboard.addPrimaryClipChangedListener(listener);
        }
        return START_STICKY;
    }

    private void offerCall(String number) {
        // ACTION_DIAL только открывает номеронабиратель — разрешение CALL_PHONE не нужно
        Intent dial = new Intent(Intent.ACTION_DIAL, Uri.parse("tel:" + Uri.encode(number)));
        PendingIntent pi = PendingIntent.getActivity(this, 0, dial, PendingIntent.FLAG_IMMUTABLE);
        getSystemService(NotificationManager.class).notify(310,
                new NotificationCompat.Builder(this, Notifs.CHANNEL)
                        .setSmallIcon(android.R.drawable.sym_action_call)
                        .setContentTitle("Позвонить?")
                        .setContentText(number)
                        .setContentIntent(pi)
                        .setAutoCancel(true)
                        .build());
    }

    @Override
    public void onDestroy() {
        if (clipboard != null) {
            clipboard.removePrimaryClipChangedListener(listener);
        }
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

---

### 10. Автоостановка Foreground-службы через 30 минут

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<service android:name=".services.TimedForegroundService" android:exported="false"
    android:foregroundServiceType="specialUse">
    <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="timed_task" />
</service>
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.util.Log;

import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

public class TimedForegroundService extends Service {
    private static final long MAX_RUN_MS = 30L * 60 * 1000; // 30 минут
    private final Handler handler = new Handler(Looper.getMainLooper());

    private final Runnable autoStop = () -> {
        Log.d("TimedService", "30 минут прошло, останавливаюсь");
        // аналог stopForeground(true): убрать уведомление вместе с foreground-статусом
        ServiceCompat.stopForeground(this, ServiceCompat.STOP_FOREGROUND_REMOVE);
        stopSelf();
    };

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 311,
                Notifs.build(this, "Фоновая работа", "Остановится через 30 минут"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE);
        handler.removeCallbacks(autoStop);
        handler.postDelayed(autoStop, MAX_RUN_MS);
        return START_NOT_STICKY; // при STICKY-перезапуске таймер начался бы заново
    }

    @Override
    public void onDestroy() {
        handler.removeCallbacks(autoStop);
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Для проверки временно поставьте `MAX_RUN_MS = 30_000`. `stopForeground(true)` помечен устаревшим, `STOP_FOREGROUND_REMOVE` делает то же самое.

---

## Блок 2. Широковещательные приёмники

### 11. Подключение/отключение зарядки + тип зарядки

В Android нет публичного флага «быстрая зарядка», поэтому тип определяем по `EXTRA_PLUGGED` (AC / USB / беспроводная), а «быструю» оцениваем приблизительно по току (`CURRENT_NOW`, у разных производителей единицы и знак отличаются, считайте это ориентиром).

```java
package com.university.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.BatteryManager;
import android.os.Bundle;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

public class ChargerActivity extends AppCompatActivity {
    private TextView view;

    private final BroadcastReceiver receiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (Intent.ACTION_POWER_CONNECTED.equals(intent.getAction())) {
                view.setText("Зарядка подключена: " + chargeType(context));
            } else if (Intent.ACTION_POWER_DISCONNECTED.equals(intent.getAction())) {
                view.setText("Зарядка отключена");
            }
        }
    };

    private static String chargeType(Context context) {
        // null-приёмник возвращает последний «липкий» интент батареи, регистрации не происходит
        Intent battery = context.registerReceiver(null, new IntentFilter(Intent.ACTION_BATTERY_CHANGED));
        int plugged = (battery == null) ? 0 : battery.getIntExtra(BatteryManager.EXTRA_PLUGGED, 0);
        String type;
        switch (plugged) {
            case BatteryManager.BATTERY_PLUGGED_AC:
                type = "от сети (AC)";
                break;
            case BatteryManager.BATTERY_PLUGGED_USB:
                type = "USB";
                break;
            case BatteryManager.BATTERY_PLUGGED_WIRELESS:
                type = "беспроводная";
                break;
            default:
                type = "неизвестный источник";
        }
        BatteryManager bm = context.getSystemService(BatteryManager.class);
        long microAmps = Math.abs(bm.getLongProperty(BatteryManager.BATTERY_PROPERTY_CURRENT_NOW));
        return type + (microAmps >= 2_000_000L ? ", похоже на быструю" : "");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        view = new TextView(this);
        view.setTextSize(22);
        view.setPadding(32, 32, 32, 32);
        view.setText("Подключите или отключите зарядку");
        setContentView(view);
    }

    @Override
    protected void onStart() {
        super.onStart();
        IntentFilter filter = new IntentFilter();
        filter.addAction(Intent.ACTION_POWER_CONNECTED);
        filter.addAction(Intent.ACTION_POWER_DISCONNECTED);
        ContextCompat.registerReceiver(this, receiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    protected void onStop() {
        super.onStop();
        unregisterReceiver(receiver);
    }
}
```

---

### 12. Смена часового пояса → пересчёт локальных будильников

Будильники хранятся как «часы:минуты по местному времени», а не как абсолютный момент. Поэтому при смене пояса их надо поставить заново. Системный `TIMEZONE_CHANGED` разрешён и в манифесте.

```xml
<receiver android:name=".receivers.TimeZoneReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.TIMEZONE_CHANGED" />
    </intent-filter>
</receiver>
<receiver android:name=".receivers.AlarmFireReceiver" android:exported="false" />
```

```java
package com.university.app.receivers;

import android.app.AlarmManager;
import android.app.PendingIntent;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.util.Log;

import java.util.Calendar;
import java.util.HashSet;
import java.util.Set;
import java.util.TimeZone;

public class TimeZoneReceiver extends BroadcastReceiver {
    private static final String PREFS = "alarms";
    private static final String KEY_SET = "alarm_set"; // элементы вида "1|07:30"

    /** Сохраняет и ставит будильник. */
    public static void addAlarm(Context ctx, int id, int hour, int minute) {
        SharedPreferences p = ctx.getSharedPreferences(PREFS, Context.MODE_PRIVATE);
        Set<String> set = new HashSet<>(p.getStringSet(KEY_SET, new HashSet<>()));
        set.add(id + "|" + hour + ":" + minute);
        p.edit().putStringSet(KEY_SET, set).apply();
        schedule(ctx, id, hour, minute);
    }

    static void schedule(Context ctx, int id, int hour, int minute) {
        Calendar c = Calendar.getInstance(); // уже в новом поясе
        c.set(Calendar.HOUR_OF_DAY, hour);
        c.set(Calendar.MINUTE, minute);
        c.set(Calendar.SECOND, 0);
        c.set(Calendar.MILLISECOND, 0);
        if (c.getTimeInMillis() <= System.currentTimeMillis()) {
            c.add(Calendar.DAY_OF_YEAR, 1);
        }
        PendingIntent fire = PendingIntent.getBroadcast(ctx, id,
                new Intent(ctx, AlarmFireReceiver.class).putExtra("id", id),
                PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_UPDATE_CURRENT);
        PendingIntent show = PendingIntent.getActivity(ctx, id,
                ctx.getPackageManager().getLaunchIntentForPackage(ctx.getPackageName()),
                PendingIntent.FLAG_IMMUTABLE);
        // setAlarmClock не требует специального разрешения на точные будильники
        ctx.getSystemService(AlarmManager.class)
                .setAlarmClock(new AlarmManager.AlarmClockInfo(c.getTimeInMillis(), show), fire);
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        if (!Intent.ACTION_TIMEZONE_CHANGED.equals(intent.getAction())) return;
        TimeZone.setDefault(null); // сбрасываем кэш пояса JVM
        Log.d("TimeZone", "Новый пояс: " + TimeZone.getDefault().getID());

        SharedPreferences p = context.getSharedPreferences(PREFS, Context.MODE_PRIVATE);
        for (String item : p.getStringSet(KEY_SET, new HashSet<>())) {
            String[] idAndTime = item.split("\\|");
            String[] hm = idAndTime[1].split(":");
            schedule(context, Integer.parseInt(idAndTime[0]),
                    Integer.parseInt(hm[0]), Integer.parseInt(hm[1]));
        }
    }
}
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class AlarmFireReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.d("Alarm", "Сработал будильник #" + intent.getIntExtra("id", -1));
    }
}
```
Проверка: `TimeZoneReceiver.addAlarm(this, 1, 7, 30);` затем в настройках эмулятора смените часовой пояс: в логе будет новый пояс, а будильник переставится.

---

### 13. Перехват исходящих SMS

Здесь нужно честно сказать: **в Android нет публичного бродкаста, который позволял бы обычному приложению перехватывать исходящие SMS других приложений**, и это сделано намеренно ради приватности. Старые обходные приёмы (наблюдение за базой SMS, `NEW_OUTGOING_SMS`) закрыты, а `READ_SMS`/`SEND_SMS` в Google Play разрешены только приложению-обработчику SMS по умолчанию. Реализовать «перехват» чужих сообщений как задумано нельзя, и я не буду подсовывать обходные пути.

Что возможно легально: отследить статус **своих** отправленных сообщений.

```xml
<uses-permission android:name="android.permission.SEND_SMS" />
<receiver android:name=".receivers.SmsSentReceiver" android:exported="false" />
```

```java
package com.university.app.receivers;

import android.app.Activity;
import android.app.PendingIntent;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.telephony.SmsManager;
import android.util.Log;

public class SmsSentReceiver extends BroadcastReceiver {
    public static final String EXTRA_TO = "to";

    /** Отправка своего SMS с отслеживанием результата. */
    public static void send(Context ctx, String to, String text) {
        Intent intent = new Intent(ctx, SmsSentReceiver.class).putExtra(EXTRA_TO, to);
        PendingIntent sent = PendingIntent.getBroadcast(ctx, to.hashCode(), intent,
                PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_UPDATE_CURRENT);
        ctx.getSystemService(SmsManager.class).sendTextMessage(to, null, text, sent, null);
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        String to = intent.getStringExtra(EXTRA_TO);
        if (getResultCode() == Activity.RESULT_OK) {
            Log.d("SmsSent", "Отправлено на " + to);
        } else {
            Log.w("SmsSent", "Ошибка отправки на " + to + ", код " + getResultCode());
        }
    }
}
```
Перед вызовом `send()` запросите `SEND_SMS` в рантайме.

---

### 14. Нехватка места: `ACTION_DEVICE_STORAGE_LOW` → сброс временных картинок

Событие устарело (с API 26 в манифест-приёмники не доставляется), но динамическая регистрация ещё работает. Регистрируем в `Application`, чтобы приёмник жил, пока жив процесс.

```java
package com.university.app;

import android.app.Application;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.util.Log;

import androidx.core.content.ContextCompat;

import java.io.File;

public class App extends Application {

    private final BroadcastReceiver storageLowReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final PendingResult pending = goAsync();
            new Thread(() -> {
                try {
                    int n = deleteAll(new File(getCacheDir(), "tmp_images"));
                    Log.w("StorageLow", "Мало места. Удалено временных картинок: " + n);
                } finally {
                    pending.finish();
                }
            }).start();
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        ContextCompat.registerReceiver(this, storageLowReceiver,
                new IntentFilter(Intent.ACTION_DEVICE_STORAGE_LOW),
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    private static int deleteAll(File dir) {
        int count = 0;
        File[] files = dir.listFiles();
        if (files == null) return 0;
        for (File f : files) {
            if (f.isDirectory()) {
                count += deleteAll(f);
            }
            if (f.delete()) count++;
        }
        return count;
    }
}
```
Не забудьте `android:name=".App"` в `<application>`. Бонус: с Android 8+ система сама чистит `getCacheDir()` при нехватке места, поэтому временные файлы кладите именно туда.

---

### 15. Автозапуск службы после перезагрузки: `BOOT_COMPLETED` → проверка настроек → `startForegroundService()`

Тип `specialUse` выбран осознанно: на Android 15 из `BOOT_COMPLETED` **запрещено** запускать сервисы типов `dataSync`, `mediaPlayback`, `camera`, `microphone`, `phoneCall`, `mediaProjection`.

```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />

<receiver android:name=".receivers.BootChainReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
<service android:name=".services.BootStartedService" android:exported="false"
    android:foregroundServiceType="specialUse">
    <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="device_monitoring" />
</service>
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.util.Log;

import androidx.core.content.ContextCompat;

import com.university.app.services.BootStartedService;

public class BootChainReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (!Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            return;
        }
        // Шаг 2: валидация настроек
        SharedPreferences prefs = context.getSharedPreferences("settings", Context.MODE_PRIVATE);
        if (!prefs.getBoolean("autostart_enabled", false)) {
            Log.d("BootChain", "Автозапуск выключен пользователем");
            return;
        }
        // Шаг 3: запуск службы
        try {
            ContextCompat.startForegroundService(context, new Intent(context, BootStartedService.class));
        } catch (IllegalStateException e) { // ForegroundServiceStartNotAllowedException — подкласс
            Log.e("BootChain", "Система не разрешила запуск сервиса", e);
        }
    }
}
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.os.IBinder;

import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

public class BootStartedService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 315,
                Notifs.build(this, "Служба запущена после загрузки", "Работаю в фоне"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE);
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Включить автозапуск: `getSharedPreferences("settings", MODE_PRIVATE).edit().putBoolean("autostart_enabled", true).apply();`

---

### 16. Подключение гарнитуры → пауза плеера

`ACTION_HEADSET_PLUG` работает только для проводных наушников и только динамически. Рекомендуемый системный способ — `ACTION_AUDIO_BECOMING_NOISY`: он приходит при отключении и проводных, и Bluetooth-наушников. Обрабатываем оба.

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.media.AudioManager;
import android.media.MediaPlayer;

public class HeadsetPauseReceiver extends BroadcastReceiver {
    private final MediaPlayer player;

    public HeadsetPauseReceiver(MediaPlayer player) {
        this.player = player;
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        boolean unplugged = AudioManager.ACTION_AUDIO_BECOMING_NOISY.equals(action)
                || (Intent.ACTION_HEADSET_PLUG.equals(action)
                && intent.getIntExtra("state", -1) == 0); // 0 = отключили
        if (unplugged && player.isPlaying()) {
            player.pause();
        }
    }
}
```

Регистрация в сервисе плеера:
```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.content.IntentFilter;
import android.media.AudioManager;
import android.media.MediaPlayer;
import android.os.IBinder;

import androidx.core.content.ContextCompat;

import com.university.app.R;
import com.university.app.receivers.HeadsetPauseReceiver;

public class HeadsetAwarePlayerService extends Service {
    private MediaPlayer player;
    private HeadsetPauseReceiver headsetReceiver;

    @Override
    public void onCreate() {
        super.onCreate();
        player = MediaPlayer.create(this, R.raw.track1);
        headsetReceiver = new HeadsetPauseReceiver(player);
        IntentFilter filter = new IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY);
        filter.addAction(Intent.ACTION_HEADSET_PLUG);
        ContextCompat.registerReceiver(this, headsetReceiver, filter,
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (player != null) player.start();
        return START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        unregisterReceiver(headsetReceiver);
        if (player != null) player.release();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
(Для работы в фоне оформите как foreground-сервис с типом `mediaPlayback`, как в задании 2.)

---

### 17. «Подписанные» внутренние бродкасты: custom permission уровня `signature`

Система сама проверяет, что оба приложения подписаны **одним ключом**, поэтому поддельное приложение отправить или принять такое сообщение не сможет. Отдельная подпись сообщения не нужна.

Манифест (в **обоих** приложениях, чтобы не зависеть от порядка установки):
```xml
<permission android:name="com.university.app.permission.INTERNAL_BROADCAST"
    android:protectionLevel="signature" />
<uses-permission android:name="com.university.app.permission.INTERNAL_BROADCAST" />

<!-- Принимающая сторона: слать сюда могут только владельцы разрешения -->
<receiver android:name=".receivers.SecureReceiver" android:exported="true"
    android:permission="com.university.app.permission.INTERNAL_BROADCAST">
    <intent-filter>
        <action android:name="com.university.app.SECURE_EVENT" />
    </intent-filter>
</receiver>
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class SecureReceiver extends BroadcastReceiver {
    public static final String ACTION = "com.university.app.SECURE_EVENT";
    public static final String PERMISSION = "com.university.app.permission.INTERNAL_BROADCAST";

    @Override
    public void onReceive(Context context, Intent intent) {
        if (!ACTION.equals(intent.getAction())) return;
        Log.d("SecureReceiver", "Доверенное сообщение: " + intent.getStringExtra("payload"));
    }

    /** Отправка: доставится только приёмникам, у которых есть это разрешение. */
    public static void send(Context ctx, String targetPackage, String payload) {
        Intent intent = new Intent(ACTION);
        intent.setPackage(targetPackage); // явный адресат (иначе неявный бродкаст в манифест не придёт)
        intent.putExtra("payload", payload);
        ctx.sendBroadcast(intent, PERMISSION);
    }
}
```
Для динамического приёмника разрешение отправителя указывается при регистрации:
`ContextCompat.registerReceiver(ctx, receiver, new IntentFilter(ACTION), SecureReceiver.PERMISSION, null, ContextCompat.RECEIVER_EXPORTED);`

---

### 18. Смена языка (`ACTION_LOCALE_CHANGED`) → сброс локализованных кэшей

```xml
<receiver android:name=".receivers.LocaleChangedReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.LOCALE_CHANGED" />
    </intent-filter>
</receiver>
```

```java
package com.university.app.util;

import java.util.Locale;
import java.util.concurrent.ConcurrentHashMap;

public final class LocalizedCache {
    private static final ConcurrentHashMap<String, String> CACHE = new ConcurrentHashMap<>();
    private static volatile Locale cachedLocale = Locale.getDefault();

    private LocalizedCache() { }

    public static String get(String key) {
        return CACHE.get(key + "@" + cachedLocale);
    }

    public static void put(String key, String value) {
        CACHE.put(key + "@" + cachedLocale, value);
    }

    public static void reset() {
        CACHE.clear();
        cachedLocale = Locale.getDefault(); // подхватываем новый язык
    }
}
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

import com.university.app.util.LocalizedCache;

import java.util.Locale;

public class LocaleChangedReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (Intent.ACTION_LOCALE_CHANGED.equals(intent.getAction())) {
            LocalizedCache.reset();
            Log.d("LocaleChanged", "Новый язык: " + Locale.getDefault() + ", кэш сброшен");
        }
    }
}
```

---

### 19. Три приёмника с приоритетами и `setResultData()`

Каждый приёмник (хоть в разных Gradle-модулях: манифесты сольются) читает результат предыдущего и дописывает свой. `setResultData()` работает **только** в упорядоченных бродкастах.

```xml
<receiver android:name=".receivers.ModuleAReceiver" android:exported="false">
    <intent-filter android:priority="300">
        <action android:name="com.university.app.PIPELINE" />
    </intent-filter>
</receiver>
<receiver android:name=".receivers.ModuleBReceiver" android:exported="false">
    <intent-filter android:priority="200">
        <action android:name="com.university.app.PIPELINE" />
    </intent-filter>
</receiver>
<receiver android:name=".receivers.ModuleCReceiver" android:exported="false">
    <intent-filter android:priority="100">
        <action android:name="com.university.app.PIPELINE" />
    </intent-filter>
</receiver>
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;

public class ModuleAReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        setResultData(getResultData() + " -> A(300)");
    }
}
```
```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;

public class ModuleBReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        setResultData(getResultData() + " -> B(200)");
    }
}
```
```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;

public class ModuleCReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        setResultData(getResultData() + " -> C(100)");
    }
}
```

Отправитель:
```java
package com.university.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.os.Bundle;
import android.util.Log;
import android.widget.Button;

import androidx.appcompat.app.AppCompatActivity;

public class PipelineActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button = new Button(this);
        button.setText("Запустить конвейер");
        button.setOnClickListener(v -> {
            Intent intent = new Intent("com.university.app.PIPELINE");
            intent.setPackage(getPackageName());
            BroadcastReceiver finalReceiver = new BroadcastReceiver() {
                @Override
                public void onReceive(Context context, Intent i) {
                    Log.d("Pipeline", "Итог: " + getResultData()); // start -> A(300) -> B(200) -> C(100)
                }
            };
            sendOrderedBroadcast(intent, null, finalReceiver, null,
                    RESULT_OK, "start", null);
        });
        setContentView(button);
    }
}
```

---

### 20. «Антикраш»-приёмник: `goAsync()` + принудительное завершение через 8 секунд

Работа идёт в одном пуле, а отдельная «сторожевая» задача ждёт её не дольше 8 секунд, отменяет и обязательно вызывает `finish()`. Сама работа должна реагировать на прерывание (`isInterrupted`/`InterruptedException`).

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class SafeReceiver extends BroadcastReceiver {
    private static final String TAG = "SafeReceiver";
    private static final ExecutorService WORKERS = Executors.newFixedThreadPool(2);
    private static final ExecutorService WATCHDOGS = Executors.newFixedThreadPool(2);

    @Override
    public void onReceive(Context context, Intent intent) {
        final PendingResult pending = goAsync();
        final Context appContext = context.getApplicationContext();

        WATCHDOGS.execute(() -> {
            Future<?> job = WORKERS.submit(() -> doWork(appContext));
            try {
                job.get(8, TimeUnit.SECONDS);
            } catch (TimeoutException e) {
                job.cancel(true); // прерываем зависшую работу
                Log.w(TAG, "Таймаут 8 с: работа прервана");
            } catch (InterruptedException e) {
                job.cancel(true);
                Thread.currentThread().interrupt();
            } catch (ExecutionException e) {
                Log.e(TAG, "Работа упала", e.getCause()); // ошибка не роняет приложение
            } finally {
                pending.finish(); // всегда сообщаем системе, что закончили
            }
        });
    }

    private void doWork(Context context) {
        try {
            Thread.sleep(2000); // имитация; поставьте 20000 и увидите таймаут
            Log.d(TAG, "Работа успешно завершена");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## Блок 3. Content Provider

### 21. Провайдер логов: только чтение, данные зашифрованы

Права: `readPermission` для чтения, для записи заведомо недоступное разрешение, а все методы записи бросают исключение. Логи шифруются AES-GCM ключом из Android Keystore.

Честное замечание: ключ из Keystore **привязан к вашему приложению**, аналитический сервис расшифровать им не сможет. В реальной схеме логи шифруют **открытым ключом** аналитики (гибридное шифрование) или передают общий ключ защищённым каналом. Здесь показана сама механика провайдера.

```xml
<permission android:name="com.university.permission.READ_LOGS" android:protectionLevel="signature" />
<permission android:name="com.university.permission.NO_WRITE_LOGS" android:protectionLevel="signature" />
<provider
    android:name=".providers.SecureLogProvider"
    android:authorities="com.university.app.logs"
    android:exported="true"
    android:readPermission="com.university.permission.READ_LOGS"
    android:writePermission="com.university.permission.NO_WRITE_LOGS" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.MatrixCursor;
import android.net.Uri;
import android.security.keystore.KeyGenParameterSpec;
import android.security.keystore.KeyProperties;
import android.util.Base64;

import java.nio.charset.StandardCharsets;
import java.security.KeyStore;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;

public class SecureLogProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.logs";
    private static final String ALIAS = "logs_key";
    private static final int CODE_LOGS = 1;
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        MATCHER.addURI(AUTHORITY, "logs", CODE_LOGS);
    }

    /** Источник логов приложения (в реальности — файл/БД). */
    public static final List<String> LOG_LINES = new CopyOnWriteArrayList<>();

    @Override
    public boolean onCreate() {
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        if (MATCHER.match(uri) != CODE_LOGS) {
            throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        MatrixCursor cursor = new MatrixCursor(new String[]{"_id", "message_enc", "iv"});
        try {
            SecretKey key = getOrCreateKey();
            int id = 0;
            for (String line : LOG_LINES) {
                Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
                cipher.init(Cipher.ENCRYPT_MODE, key);
                byte[] enc = cipher.doFinal(line.getBytes(StandardCharsets.UTF_8));
                cursor.addRow(new Object[]{id++,
                        Base64.encodeToString(enc, Base64.NO_WRAP),
                        Base64.encodeToString(cipher.getIV(), Base64.NO_WRAP)});
            }
        } catch (Exception e) {
            throw new IllegalStateException("Ошибка шифрования логов", e);
        }
        return cursor;
    }

    private SecretKey getOrCreateKey() throws Exception {
        KeyStore ks = KeyStore.getInstance("AndroidKeyStore");
        ks.load(null);
        if (!ks.containsAlias(ALIAS)) {
            KeyGenerator gen = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore");
            gen.init(new KeyGenParameterSpec.Builder(ALIAS,
                    KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT)
                    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                    .setKeySize(256)
                    .build());
            return gen.generateKey();
        }
        return (SecretKey) ks.getKey(ALIAS, null);
    }

    @Override
    public String getType(Uri uri) {
        return "vnd.android.cursor.dir/vnd.com.university.log";
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        throw new UnsupportedOperationException("Провайдер только для чтения");
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        throw new UnsupportedOperationException("Провайдер только для чтения");
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        throw new UnsupportedOperationException("Провайдер только для чтения");
    }
}
```

---

### 22. Поисковые подсказки (`SUGGEST_URI_PATH_QUERY`)

Система запрашивает `content://<authority>/search_suggest_query/<введённый текст>`, а мы возвращаем курсор со стандартными колонками. Подсказки показываются в `SearchView` вашего приложения. (Глобальная строка поиска Quick Search Box на современных Android практически не используется.)

`res/xml/searchable.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
    android:label="@string/app_name"
    android:hint="Поиск города"
    android:searchSuggestAuthority="com.university.app.suggest"
    android:searchSuggestIntentAction="android.intent.action.VIEW" />
```

Манифест:
```xml
<provider
    android:name=".providers.SuggestProvider"
    android:authorities="com.university.app.suggest"
    android:exported="true" />

<activity android:name=".SearchResultActivity" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.SEARCH" />
    </intent-filter>
    <meta-data android:name="android.app.searchable" android:resource="@xml/searchable" />
</activity>
```

```java
package com.university.app.providers;

import android.app.SearchManager;
import android.content.ContentProvider;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.MatrixCursor;
import android.net.Uri;
import android.provider.BaseColumns;

import java.util.Locale;

public class SuggestProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.suggest";
    private static final int CODE_SUGGEST = 1;
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);
    private static final String[] CITIES = {"Тула", "Москва", "Тверь", "Томск", "Тюмень", "Казань", "Калуга"};

    static {
        MATCHER.addURI(AUTHORITY, SearchManager.SUGGEST_URI_PATH_QUERY, CODE_SUGGEST);
        MATCHER.addURI(AUTHORITY, SearchManager.SUGGEST_URI_PATH_QUERY + "/*", CODE_SUGGEST);
    }

    @Override
    public boolean onCreate() {
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        if (MATCHER.match(uri) != CODE_SUGGEST) {
            throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        String last = uri.getLastPathSegment();
        String query = SearchManager.SUGGEST_URI_PATH_QUERY.equals(last)
                ? "" : last.toLowerCase(Locale.getDefault());

        MatrixCursor cursor = new MatrixCursor(new String[]{
                BaseColumns._ID,
                SearchManager.SUGGEST_COLUMN_TEXT_1,
                SearchManager.SUGGEST_COLUMN_INTENT_DATA});
        int id = 0;
        for (String city : CITIES) {
            if (city.toLowerCase(Locale.getDefault()).startsWith(query) && id < 10) {
                cursor.addRow(new Object[]{id++, city, city});
            }
        }
        return cursor;
    }

    @Override
    public String getType(Uri uri) {
        return SearchManager.SUGGEST_MIME_TYPE;
    }

    @Override public Uri insert(Uri uri, ContentValues values) { return null; }
    @Override public int delete(Uri uri, String selection, String[] selectionArgs) { return 0; }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { return 0; }
}
```

---

### 23. Экспорт визитных карточек (`.vcf`) через `openFile()`

```xml
<provider
    android:name=".providers.VCardProvider"
    android:authorities="com.university.app.vcard"
    android:exported="false"
    android:grantUriPermissions="true" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.MatrixCursor;
import android.net.Uri;
import android.os.ParcelFileDescriptor;
import android.provider.OpenableColumns;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class VCardProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.vcard";
    private static final int CODE_CONTACT = 1;
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);

    // id -> {имя, фамилия, телефон, email}
    private static final String[][] CONTACTS = {
            {"Иван", "Петров", "+79001112233", "ivan@example.com"},
            {"Анна", "Сидорова", "+79004445566", "anna@example.com"}
    };

    static {
        MATCHER.addURI(AUTHORITY, "contact/#", CODE_CONTACT);
    }

    @Override
    public boolean onCreate() {
        return true;
    }

    @Override
    public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
        if (MATCHER.match(uri) != CODE_CONTACT) {
            throw new FileNotFoundException("Неизвестный URI: " + uri);
        }
        if (!"r".equals(mode)) {
            throw new SecurityException("Только чтение");
        }
        int id = (int) android.content.ContentUris.parseId(uri);
        if (id < 0 || id >= CONTACTS.length) {
            throw new FileNotFoundException("Нет контакта " + id);
        }
        File dir = new File(getContext().getCacheDir(), "vcards");
        if (!dir.exists() && !dir.mkdirs()) {
            throw new FileNotFoundException("Нет каталога кэша");
        }
        File file = new File(dir, "contact_" + id + ".vcf");
        try (FileOutputStream out = new FileOutputStream(file)) {
            out.write(buildVCard(CONTACTS[id]).getBytes(StandardCharsets.UTF_8));
        } catch (IOException e) {
            throw new FileNotFoundException("Не удалось подготовить vCard");
        }
        return ParcelFileDescriptor.open(file, ParcelFileDescriptor.MODE_READ_ONLY);
    }

    private static String buildVCard(String[] c) {
        return "BEGIN:VCARD\r\n"
                + "VERSION:3.0\r\n"
                + "N:" + esc(c[1]) + ";" + esc(c[0]) + ";;;\r\n"
                + "FN:" + esc(c[0]) + " " + esc(c[1]) + "\r\n"
                + "TEL;TYPE=CELL:" + esc(c[2]) + "\r\n"
                + "EMAIL:" + esc(c[3]) + "\r\n"
                + "END:VCARD\r\n";
    }

    // Экранирование служебных символов vCard
    private static String esc(String s) {
        return s.replace("\\", "\\\\").replace(",", "\\,").replace(";", "\\;").replace("\n", "\\n");
    }

    // Приложения-получатели спрашивают имя и размер файла
    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        int id = (int) android.content.ContentUris.parseId(uri);
        MatrixCursor c = new MatrixCursor(new String[]{OpenableColumns.DISPLAY_NAME, OpenableColumns.SIZE});
        c.addRow(new Object[]{"contact_" + id + ".vcf", null});
        return c;
    }

    @Override
    public String getType(Uri uri) {
        return "text/x-vcard";
    }

    @Override public Uri insert(Uri uri, ContentValues values) { return null; }
    @Override public int delete(Uri uri, String selection, String[] selectionArgs) { return 0; }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { return 0; }
}
```

Передача другому приложению:
```java
Uri uri = Uri.parse("content://com.university.app.vcard/contact/0");
Intent send = new Intent(Intent.ACTION_SEND);
send.setType("text/x-vcard");
send.putExtra(Intent.EXTRA_STREAM, uri);
send.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION); // временный доступ получателю
startActivity(Intent.createChooser(send, "Отправить контакт"));
```

---

### 24. Синхронизация корзины между двумя приложениями

**Приложение A (магазин, владелец данных)** — провайдер на SQLite:

```xml
<!-- Манифест A и B (одинаково), одна подпись у обоих приложений -->
<permission android:name="com.company.cart.permission.READ_CART" android:protectionLevel="signature" />
<uses-permission android:name="com.company.cart.permission.READ_CART" />

<!-- Только в A -->
<provider
    android:name=".providers.CartProvider"
    android:authorities="com.company.shop.cart"
    android:exported="true"
    android:readPermission="com.company.cart.permission.READ_CART" />
```

```java
package com.company.shop.providers;

import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.Context;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.net.Uri;

public class CartProvider extends ContentProvider {
    public static final String AUTHORITY = "com.company.shop.cart";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/items");
    private static final int CODE_ITEMS = 1;
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        MATCHER.addURI(AUTHORITY, "items", CODE_ITEMS);
    }

    private static class Db extends SQLiteOpenHelper {
        Db(Context c) {
            super(c, "cart.db", null, 1);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE items (_id INTEGER PRIMARY KEY AUTOINCREMENT,"
                    + "product TEXT NOT NULL, qty INTEGER NOT NULL, price REAL NOT NULL)");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int o, int n) {
            db.execSQL("DROP TABLE IF EXISTS items");
            onCreate(db);
        }
    }

    private Db helper;

    @Override
    public boolean onCreate() {
        helper = new Db(getContext());
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        check(uri);
        Cursor c = helper.getReadableDatabase().query("items", projection, selection, selectionArgs,
                null, null, sortOrder);
        c.setNotificationUri(getContext().getContentResolver(), uri);
        return c;
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        check(uri);
        long id = helper.getWritableDatabase().insertOrThrow("items", null, values);
        getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return ContentUris.withAppendedId(CONTENT_URI, id);
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] args) {
        check(uri);
        int n = helper.getWritableDatabase().update("items", values, selection, args);
        if (n > 0) getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return n;
    }

    @Override
    public int delete(Uri uri, String selection, String[] args) {
        check(uri);
        int n = helper.getWritableDatabase().delete("items", selection, args);
        if (n > 0) getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return n;
    }

    @Override
    public String getType(Uri uri) {
        return "vnd.android.cursor.dir/vnd.com.company.cart";
    }

    private void check(Uri uri) {
        if (MATCHER.match(uri) != CODE_ITEMS) {
            throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
    }
}
```

**Приложение B (читает корзину A)**. С Android 11 нужно объявить видимость провайдера:

```xml
<queries>
    <provider android:authorities="com.company.shop.cart" />
</queries>
```

```java
package com.company.partner;

import android.content.ContentResolver;
import android.database.Cursor;
import android.net.Uri;

public class CartReader {
    private static final Uri CART_URI = Uri.parse("content://com.company.shop.cart/items");

    /** Возвращает текст корзины или null, если приложение A не установлено / нет доступа. */
    public static String readCart(ContentResolver resolver) {
        try (Cursor c = resolver.query(CART_URI,
                new String[]{"product", "qty", "price"}, null, null, "_id ASC")) {
            if (c == null) {
                return null; // провайдера нет
            }
            StringBuilder sb = new StringBuilder();
            double total = 0;
            while (c.moveToNext()) {
                String name = c.getString(0);
                int qty = c.getInt(1);
                double price = c.getDouble(2);
                total += qty * price;
                sb.append(name).append(" × ").append(qty).append(" = ").append(qty * price).append('\n');
            }
            return sb.append("Итого: ").append(total).toString();
        } catch (SecurityException e) {
            return null; // разные подписи или нет разрешения
        }
    }
}
```
Почему разрешение объявлено в обоих манифестах: `signature`-разрешение выдаётся только при совпадении подписи, и если B поставили раньше A, без собственного объявления оно могло бы не выдаться.

---

### 25. Провайдер настроек на `EncryptedSharedPreferences`

Зависимость: `implementation 'androidx.security:security-crypto:1.1.0-alpha06'`. Замечание: эта библиотека объявлена устаревшей (поддерживается, но новые функции не добавляются); для нового кода часто берут DataStore + Keystore. Для задания она подходит.

Инициализация вынесена из `onCreate()` в ленивый метод, потому что работа с Keystore медленная, а `onCreate()` блокирует старт приложения.

```xml
<provider
    android:name=".providers.ConfigProvider"
    android:authorities="com.university.app.config"
    android:exported="true"
    android:readPermission="com.university.permission.READ_CONFIG"
    android:writePermission="com.university.permission.WRITE_CONFIG" />
<permission android:name="com.university.permission.READ_CONFIG" android:protectionLevel="signature" />
<permission android:name="com.university.permission.WRITE_CONFIG" android:protectionLevel="signature" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.SharedPreferences;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.MatrixCursor;
import android.net.Uri;

import androidx.security.crypto.EncryptedSharedPreferences;
import androidx.security.crypto.MasterKey;

import java.io.IOException;
import java.security.GeneralSecurityException;
import java.util.Map;

public class ConfigProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.config";
    private static final int CODE_ALL = 1;
    private static final int CODE_KEY = 2;
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        MATCHER.addURI(AUTHORITY, "config", CODE_ALL);
        MATCHER.addURI(AUTHORITY, "config/*", CODE_KEY);
    }

    private SharedPreferences prefs;

    @Override
    public boolean onCreate() {
        return true; // ничего тяжёлого при старте
    }

    private synchronized SharedPreferences prefs() {
        if (prefs == null) {
            try {
                MasterKey key = new MasterKey.Builder(getContext())
                        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build();
                prefs = EncryptedSharedPreferences.create(getContext(), "secure_config", key,
                        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
                        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM);
            } catch (GeneralSecurityException | IOException e) {
                throw new IllegalStateException("Не удалось открыть защищённое хранилище", e);
            }
        }
        return prefs;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        MatrixCursor cursor = new MatrixCursor(new String[]{"key", "value"});
        switch (MATCHER.match(uri)) {
            case CODE_ALL:
                for (Map.Entry<String, ?> e : prefs().getAll().entrySet()) {
                    cursor.addRow(new Object[]{e.getKey(), String.valueOf(e.getValue())});
                }
                break;
            case CODE_KEY:
                String key = uri.getLastPathSegment();
                if (prefs().contains(key)) {
                    cursor.addRow(new Object[]{key, prefs().getString(key, null)});
                }
                break;
            default:
                throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        cursor.setNotificationUri(getContext().getContentResolver(), uri);
        return cursor;
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        if (MATCHER.match(uri) != CODE_ALL) {
            throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        String key = values.getAsString("key");
        String value = values.getAsString("value");
        if (key == null || key.isEmpty() || value == null) {
            throw new IllegalArgumentException("Нужны поля key и value");
        }
        prefs().edit().putString(key, value).apply();
        getContext().getContentResolver().notifyChange(uri, null);
        return Uri.withAppendedPath(uri, key);
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        if (MATCHER.match(uri) != CODE_KEY || values == null || !values.containsKey("value")) {
            throw new IllegalArgumentException("Ожидается config/<key> и поле value");
        }
        prefs().edit().putString(uri.getLastPathSegment(), values.getAsString("value")).apply();
        getContext().getContentResolver().notifyChange(uri, null);
        return 1;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        if (MATCHER.match(uri) != CODE_KEY) {
            throw new IllegalArgumentException("Ожидается config/<key>");
        }
        boolean existed = prefs().contains(uri.getLastPathSegment());
        prefs().edit().remove(uri.getLastPathSegment()).apply();
        if (existed) getContext().getContentResolver().notifyChange(uri, null);
        return existed ? 1 : 0;
    }

    @Override
    public String getType(Uri uri) {
        return MATCHER.match(uri) == CODE_KEY
                ? "vnd.android.cursor.item/vnd.com.university.config"
                : "vnd.android.cursor.dir/vnd.com.university.config";
    }
}
```


---

# Часть 5. Смешанные задания 26–50

Помощник `Notifs` (создайте, если его ещё нет; он же есть в части 4):

```java
package com.university.app.util;

import android.app.Notification;
import android.content.Context;

import androidx.core.app.NotificationChannelCompat;
import androidx.core.app.NotificationCompat;
import androidx.core.app.NotificationManagerCompat;

public final class Notifs {
    public static final String CHANNEL = "system_components_channel";

    private Notifs() { }

    public static void ensureChannel(Context c) {
        NotificationManagerCompat.from(c).createNotificationChannel(
                new NotificationChannelCompat.Builder(CHANNEL, NotificationManagerCompat.IMPORTANCE_LOW)
                        .setName("Системные компоненты").build());
    }

    public static Notification build(Context c, String title, String text) {
        ensureChannel(c);
        return new NotificationCompat.Builder(c, CHANNEL)
                .setContentTitle(title)
                .setContentText(text)
                .setSmallIcon(android.R.drawable.stat_notify_sync)
                .setOngoing(true)
                .build();
    }
}
```

Файл `res/xml/file_paths.xml`, который используют задания 26 и 49:
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <files-path name="photos" path="photos/" />
</paths>
```
И провайдер в манифесте (один на приложение):
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

---

## Блок 3 (продолжение). Content Provider

### 26. Снимок с камеры через `FileProvider`

Камера пишет полноразмерное фото прямо в наш файл по `content://` URI. Разрешение `CAMERA` **не объявляйте** (иначе его придётся запрашивать): системная камера работает без него.

```xml
<uses-feature android:name="android.hardware.camera" android:required="false" />
<queries>
    <intent><action android:name="android.media.action.IMAGE_CAPTURE" /></intent>
</queries>
```

```java
package com.university.app;

import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.net.Uri;
import android.os.Bundle;
import android.widget.Button;
import android.widget.ImageView;
import android.widget.LinearLayout;
import android.widget.Toast;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;

public class CameraActivity extends AppCompatActivity {
    private Uri photoUri;
    private ImageView image;

    private final ActivityResultLauncher<Uri> takePicture =
            registerForActivityResult(new ActivityResultContracts.TakePicture(), success -> {
                if (success && photoUri != null) {
                    showScaled(photoUri);
                } else {
                    Toast.makeText(this, "Снимок не сделан", Toast.LENGTH_SHORT).show();
                }
            });

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LinearLayout root = new LinearLayout(this);
        root.setOrientation(LinearLayout.VERTICAL);
        root.setPadding(32, 32, 32, 32);
        Button shoot = new Button(this);
        shoot.setText("Сделать фото");
        image = new ImageView(this);
        shoot.setOnClickListener(v -> {
            try {
                File dir = new File(getFilesDir(), "photos");
                if (!dir.exists() && !dir.mkdirs()) {
                    throw new IOException("Нет каталога");
                }
                File file = File.createTempFile("IMG_", ".jpg", dir);
                photoUri = FileProvider.getUriForFile(this, getPackageName() + ".fileprovider", file);
                takePicture.launch(photoUri); // камера запишет в этот URI
            } catch (IOException e) {
                Toast.makeText(this, "Не удалось создать файл", Toast.LENGTH_SHORT).show();
            }
        });
        root.addView(shoot);
        root.addView(image);
        setContentView(root);
    }

    // Полноразмерное фото нельзя грузить в ImageView как есть (риск OutOfMemory)
    private void showScaled(Uri uri) {
        BitmapFactory.Options opts = new BitmapFactory.Options();
        opts.inSampleSize = 4;
        try (InputStream in = getContentResolver().openInputStream(uri)) {
            Bitmap bmp = BitmapFactory.decodeStream(in, null, opts);
            image.setImageBitmap(bmp);
        } catch (IOException e) {
            Toast.makeText(this, "Не удалось прочитать снимок", Toast.LENGTH_SHORT).show();
        }
    }
}
```

---

### 27. Быстрый `bulkInsert()` на 10 000 строк

Три приёма: одна транзакция, один подготовленный `SQLiteStatement` (запрос парсится один раз), одно уведомление в конце. Это обычно ускоряет вставку в десятки раз.

```xml
<provider android:name=".providers.ProductsProvider"
    android:authorities="com.university.app.products" android:exported="false" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.database.sqlite.SQLiteStatement;
import android.net.Uri;

public class ProductsProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.products";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/products");

    private static class Db extends SQLiteOpenHelper {
        Db(Context c) {
            super(c, "products.db", null, 1);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE products (_id INTEGER PRIMARY KEY AUTOINCREMENT,"
                    + "name TEXT NOT NULL, price REAL NOT NULL)");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int o, int n) {
            db.execSQL("DROP TABLE IF EXISTS products");
            onCreate(db);
        }
    }

    private Db helper;

    @Override
    public boolean onCreate() {
        helper = new Db(getContext());
        return true;
    }

    @Override
    public int bulkInsert(Uri uri, ContentValues[] values) {
        SQLiteDatabase db = helper.getWritableDatabase();
        int inserted = 0;
        db.beginTransaction();
        try {
            SQLiteStatement stmt = db.compileStatement("INSERT INTO products(name, price) VALUES (?, ?)");
            try {
                for (ContentValues v : values) {
                    String name = v.getAsString("name");
                    Double price = v.getAsDouble("price");
                    if (name == null || price == null) {
                        throw new IllegalArgumentException("Нужны name и price");
                    }
                    stmt.clearBindings();
                    stmt.bindString(1, name);
                    stmt.bindDouble(2, price);
                    stmt.executeInsert();
                    inserted++;
                }
            } finally {
                stmt.close();
            }
            db.setTransactionSuccessful();
        } finally {
            db.endTransaction();
        }
        getContext().getContentResolver().notifyChange(CONTENT_URI, null); // один раз, а не 10 000
        return inserted;
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        long id = helper.getWritableDatabase().insertOrThrow("products", null, values);
        getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return ContentUris.withAppendedId(CONTENT_URI, id);
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        Cursor c = helper.getReadableDatabase().query("products", projection, selection,
                selectionArgs, null, null, sortOrder);
        c.setNotificationUri(getContext().getContentResolver(), uri);
        return c;
    }

    @Override public String getType(Uri uri) { return "vnd.android.cursor.dir/vnd.com.university.product"; }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { throw new UnsupportedOperationException(); }
    @Override public int delete(Uri uri, String s, String[] a) { throw new UnsupportedOperationException(); }
}
```

Замер на клиенте:
```java
ContentValues[] batch = new ContentValues[10_000];
for (int i = 0; i < batch.length; i++) {
    ContentValues v = new ContentValues();
    v.put("name", "Товар " + i);
    v.put("price", 10 + i % 100);
    batch[i] = v;
}
long t = android.os.SystemClock.elapsedRealtime();
int n = getContentResolver().bulkInsert(ProductsProvider.CONTENT_URI, batch);
android.util.Log.d("Bulk", n + " строк за " + (android.os.SystemClock.elapsedRealtime() - t) + " мс");
```
Запускайте в фоновом потоке. Для сравнения замерьте цикл из 10 000 отдельных `insert()`.

---

### 28. Провайдер отчётов в нескольких форматах: `getStreamTypes()`

Клиент выбирает формат (`application/pdf` или `text/csv`), а провайдер генерирует данные «на лету» и отдаёт их через канал.

```xml
<provider android:name=".providers.ReportProvider"
    android:authorities="com.university.app.reports" android:exported="false"
    android:grantUriPermissions="true" />
```

```java
package com.university.app.providers;

import android.content.ClipDescription;
import android.content.ContentProvider;
import android.content.ContentValues;
import android.content.UriMatcher;
import android.content.res.AssetFileDescriptor;
import android.database.Cursor;
import android.graphics.Paint;
import android.graphics.pdf.PdfDocument;
import android.net.Uri;
import android.os.Bundle;
import android.os.ParcelFileDescriptor;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.OutputStream;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class ReportProvider extends ContentProvider
        implements ContentProvider.PipeDataWriter<String> {

    public static final String AUTHORITY = "com.university.app.reports";
    private static final int CODE_REPORT = 1;
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);
    private static final String[] TYPES = {"application/pdf", "text/csv"};
    private static final String[][] ROWS = {
            {"Месяц", "Выручка"}, {"Январь", "100"}, {"Февраль", "120"}, {"Март", "150"}
    };

    static {
        MATCHER.addURI(AUTHORITY, "report/*", CODE_REPORT);
    }

    @Override
    public boolean onCreate() {
        return true;
    }

    /** Какие форматы можем отдать по этому URI с учётом фильтра клиента. */
    @Override
    public String[] getStreamTypes(Uri uri, String mimeTypeFilter) {
        if (MATCHER.match(uri) != CODE_REPORT) {
            return null;
        }
        List<String> result = new ArrayList<>();
        for (String type : TYPES) {
            if (ClipDescription.compareMimeTypes(type, mimeTypeFilter)) {
                result.add(type);
            }
        }
        return result.isEmpty() ? null : result.toArray(new String[0]);
    }

    @Override
    public AssetFileDescriptor openTypedAssetFile(Uri uri, String mimeTypeFilter, Bundle opts)
            throws FileNotFoundException {
        String[] types = getStreamTypes(uri, mimeTypeFilter);
        if (types == null) {
            throw new FileNotFoundException("Формат не поддерживается: " + mimeTypeFilter);
        }
        ParcelFileDescriptor pipe = openPipeHelper(uri, types[0], opts, uri.getLastPathSegment(), this);
        return new AssetFileDescriptor(pipe, 0, AssetFileDescriptor.UNKNOWN_LENGTH);
    }

    /** Работает в отдельном потоке: пишем данные в канал. */
    @Override
    public void writeDataToPipe(ParcelFileDescriptor output, Uri uri, String mimeType,
                                Bundle opts, String reportName) {
        try (OutputStream out = new ParcelFileDescriptor.AutoCloseOutputStream(output)) {
            if ("application/pdf".equals(mimeType)) {
                writePdf(out, reportName);
            } else {
                StringBuilder sb = new StringBuilder();
                for (String[] row : ROWS) {
                    sb.append(String.join(";", row)).append('\n');
                }
                out.write(sb.toString().getBytes(StandardCharsets.UTF_8));
            }
        } catch (IOException e) {
            // клиент закрыл поток раньше времени — это нормально
        }
    }

    private void writePdf(OutputStream out, String title) throws IOException {
        PdfDocument doc = new PdfDocument();
        try {
            PdfDocument.Page page = doc.startPage(new PdfDocument.PageInfo.Builder(300, 400, 1).create());
            Paint paint = new Paint();
            paint.setTextSize(14f);
            page.getCanvas().drawText("Отчёт: " + title, 30, 40, paint);
            float y = 80;
            for (String[] row : ROWS) {
                page.getCanvas().drawText(row[0] + "  —  " + row[1], 30, y, paint);
                y += 24;
            }
            doc.finishPage(page);
            doc.writeTo(out);
        } finally {
            doc.close();
        }
    }

    @Override public String getType(Uri uri) { return "vnd.android.cursor.item/vnd.com.university.report"; }
    @Override public Cursor query(Uri u, String[] p, String s, String[] a, String o) { return null; }
    @Override public Uri insert(Uri uri, ContentValues values) { return null; }
    @Override public int delete(Uri uri, String s, String[] a) { return 0; }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { return 0; }
}
```

Клиент:
```java
Uri uri = Uri.parse("content://com.university.app.reports/report/q1");
String[] available = getContentResolver().getStreamTypes(uri, "*/*"); // [pdf, csv]
try (AssetFileDescriptor afd = getContentResolver().openTypedAssetFileDescriptor(uri, "text/csv", null);
     java.io.InputStream in = afd.createInputStream()) {
    byte[] data = in.readAllBytes(); // API 33+; на старых читайте циклом
    android.util.Log.d("Report", new String(data, java.nio.charset.StandardCharsets.UTF_8));
} catch (Exception e) {
    android.util.Log.e("Report", "Ошибка", e);
}
```

---

### 29. `ContentObserver` на журнал звонков + статистика

`READ_CALL_LOG` относится к ограниченным разрешениям Google Play (нужна декларация), для учебного проекта запрос в рантайме достаточен. Чтобы не терять события, запоминаем время последней обработанной записи и читаем всё новее.

```xml
<uses-permission android:name="android.permission.READ_CALL_LOG" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<service android:name=".services.CallLogWatchService" android:exported="false"
    android:foregroundServiceType="specialUse">
    <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="call_stats" />
</service>
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.pm.ServiceInfo;
import android.database.ContentObserver;
import android.database.Cursor;
import android.net.Uri;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.provider.CallLog;
import android.util.Log;

import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CallLogWatchService extends Service {
    private static final String TAG = "CallLogWatch";
    private static final String PREFS = "call_stats";

    private ExecutorService executor;
    private ContentObserver observer;

    @Override
    public void onCreate() {
        super.onCreate();
        executor = Executors.newSingleThreadExecutor();
        observer = new ContentObserver(new Handler(Looper.getMainLooper())) {
            @Override
            public void onChange(boolean selfChange, Uri uri) {
                executor.execute(CallLogWatchService.this::processNewCalls); // тяжёлое — не в главном потоке
            }
        };
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 329,
                Notifs.build(this, "Статистика звонков", "Отслеживаю журнал"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE);
        SharedPreferences p = getSharedPreferences(PREFS, Context.MODE_PRIVATE);
        if (!p.contains("last_seen")) {
            p.edit().putLong("last_seen", System.currentTimeMillis()).apply(); // старые звонки не считаем
        }
        getContentResolver().registerContentObserver(CallLog.Calls.CONTENT_URI, true, observer);
        return START_STICKY;
    }

    private void processNewCalls() {
        SharedPreferences p = getSharedPreferences(PREFS, Context.MODE_PRIVATE);
        long lastSeen = p.getLong("last_seen", 0);
        int incoming = p.getInt("incoming", 0);
        int outgoing = p.getInt("outgoing", 0);
        int missed = p.getInt("missed", 0);
        long seconds = p.getLong("seconds", 0);
        long newest = lastSeen;

        try (Cursor c = getContentResolver().query(CallLog.Calls.CONTENT_URI,
                new String[]{CallLog.Calls.TYPE, CallLog.Calls.DURATION, CallLog.Calls.DATE},
                CallLog.Calls.DATE + " > ?", new String[]{String.valueOf(lastSeen)},
                CallLog.Calls.DATE + " ASC")) {
            if (c == null) return;
            while (c.moveToNext()) {
                int type = c.getInt(0);
                seconds += c.getLong(1);
                newest = Math.max(newest, c.getLong(2));
                switch (type) {
                    case CallLog.Calls.INCOMING_TYPE:
                        incoming++;
                        break;
                    case CallLog.Calls.OUTGOING_TYPE:
                        outgoing++;
                        break;
                    case CallLog.Calls.MISSED_TYPE:
                        missed++;
                        break;
                    default:
                        break;
                }
            }
        } catch (SecurityException e) {
            Log.e(TAG, "Нет разрешения READ_CALL_LOG", e);
            return;
        }
        p.edit().putInt("incoming", incoming).putInt("outgoing", outgoing).putInt("missed", missed)
                .putLong("seconds", seconds).putLong("last_seen", newest).apply();
        Log.d(TAG, "Входящих " + incoming + ", исходящих " + outgoing
                + ", пропущенных " + missed + ", всего секунд " + seconds);
    }

    @Override
    public void onDestroy() {
        getContentResolver().unregisterContentObserver(observer);
        executor.shutdownNow();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
В `sortOrder` не используем `LIMIT` (на Android 11+ запрещено), поэтому фильтр по дате.

---

### 30. Полнотекстовый поиск (SQLite FTS4)

Важная деталь для русского языка: токенайзер по умолчанию (`simple`) приводит к нижнему регистру только латиницу, поэтому для кириллицы указываем `tokenize=unicode61`. Пользовательский текст превращаем в безопасный запрос `"слово"*` (префиксный поиск), чтобы спецсимволы FTS не ломали запрос.

```xml
<provider android:name=".providers.NotesSearchProvider"
    android:authorities="com.university.app.notes.fts" android:exported="false" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.Context;
import android.content.UriMatcher;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.net.Uri;

import java.util.ArrayList;
import java.util.List;

public class NotesSearchProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.notes.fts";
    public static final Uri NOTES_URI = Uri.parse("content://" + AUTHORITY + "/notes");
    private static final int CODE_NOTES = 1;
    private static final int CODE_SEARCH = 2;
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        MATCHER.addURI(AUTHORITY, "notes", CODE_NOTES);
        MATCHER.addURI(AUTHORITY, "search/*", CODE_SEARCH);
    }

    private static class Db extends SQLiteOpenHelper {
        Db(Context c) {
            super(c, "notes_fts.db", null, 1);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE VIRTUAL TABLE notes_fts USING fts4(title, body, tokenize=unicode61)");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int o, int n) {
            db.execSQL("DROP TABLE IF EXISTS notes_fts");
            onCreate(db);
        }
    }

    private Db helper;

    @Override
    public boolean onCreate() {
        helper = new Db(getContext());
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        SQLiteDatabase db = helper.getReadableDatabase();
        Cursor c;
        switch (MATCHER.match(uri)) {
            case CODE_SEARCH:
                String match = toMatchQuery(uri.getLastPathSegment());
                if (match.isEmpty()) {
                    throw new IllegalArgumentException("Пустой поисковый запрос");
                }
                c = db.rawQuery("SELECT docid AS _id, title, snippet(notes_fts) AS snippet "
                        + "FROM notes_fts WHERE notes_fts MATCH ?", new String[]{match});
                break;
            case CODE_NOTES:
                c = db.rawQuery("SELECT docid AS _id, title, body FROM notes_fts ORDER BY docid DESC", null);
                break;
            default:
                throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        c.setNotificationUri(getContext().getContentResolver(), NOTES_URI);
        return c;
    }

    /** "тул кал" -> "тул"* "кал"*  (слова выделяем по буквам и цифрам, кавычки в ввод не попадут). */
    private static String toMatchQuery(String raw) {
        List<String> parts = new ArrayList<>();
        for (String word : raw.split("[^\\p{L}\\p{N}]+")) {
            if (!word.isEmpty()) {
                parts.add("\"" + word + "\"*");
            }
        }
        return String.join(" ", parts); // слова связываются через AND
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        if (MATCHER.match(uri) != CODE_NOTES) {
            throw new IllegalArgumentException("Неизвестный URI: " + uri);
        }
        long id = helper.getWritableDatabase().insertOrThrow("notes_fts", null, values);
        getContext().getContentResolver().notifyChange(NOTES_URI, null);
        return ContentUris.withAppendedId(NOTES_URI, id);
    }

    @Override
    public String getType(Uri uri) {
        return "vnd.android.cursor.dir/vnd.com.university.note";
    }

    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { throw new UnsupportedOperationException(); }
    @Override public int delete(Uri uri, String s, String[] a) { throw new UnsupportedOperationException(); }
}
```

Проверка:
```java
ContentValues v = new ContentValues();
v.put("title", "Покупки");
v.put("body", "Купить хлеб и молоко");
getContentResolver().insert(NotesSearchProvider.NOTES_URI, v);

Uri search = Uri.withAppendedPath(NotesSearchProvider.NOTES_URI.buildUpon().path("search").build(), "мол");
try (Cursor c = getContentResolver().query(search, null, null, null, null)) {
    while (c != null && c.moveToNext()) {
        android.util.Log.d("FTS", c.getString(1) + ": " + c.getString(2));
    }
}
```

---

## Блок 4. Архитектурные комбинации

### 31. «Фоновый загрузчик»: Activity → Service → Broadcast → Receiver

Один упорядоченный бродкаст обслуживает два случая. Если экран открыт, его динамический приёмник (высокий приоритет) обновляет UI и обрывает цепочку. Если экран закрыт, срабатывает статический приёмник и показывает уведомление.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service android:name=".services.LoaderService" android:exported="false"
    android:foregroundServiceType="dataSync" />
<receiver android:name=".receivers.DownloadDoneReceiver" android:exported="false" />
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.os.IBinder;

import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

public class LoaderService extends Service {
    public static final String ACTION_DONE = "com.university.app.DOWNLOAD_DONE";
    public static final String EXTRA_RESULT = "result";

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 331,
                Notifs.build(this, "Загрузка", "Идёт скачивание..."),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC);
        new Thread(() -> {
            try {
                Thread.sleep(4000); // имитация скачивания
            } catch (InterruptedException ignored) {
            }
            Intent done = new Intent(ACTION_DONE);
            done.setPackage(getPackageName());
            done.putExtra(EXTRA_RESULT, "Файл успешно загружен");
            sendOrderedBroadcast(done, null); // упорядоченный, чтобы Activity могла «забрать» событие
            ServiceCompat.stopForeground(this, ServiceCompat.STOP_FOREGROUND_REMOVE);
            stopSelf(startId);
        }, "loader").start();
        return START_NOT_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

```java
package com.university.app.receivers;

import android.app.NotificationManager;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;

import androidx.core.app.NotificationCompat;

import com.university.app.services.LoaderService;
import com.university.app.util.Notifs;

public class DownloadDoneReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        // Сюда мы попадаем, только если Activity не перехватила бродкаст (экран закрыт)
        Notifs.ensureChannel(context);
        context.getSystemService(NotificationManager.class).notify(332,
                new NotificationCompat.Builder(context, Notifs.CHANNEL)
                        .setSmallIcon(android.R.drawable.stat_sys_download_done)
                        .setContentTitle("Загрузка завершена")
                        .setContentText(intent.getStringExtra(LoaderService.EXTRA_RESULT))
                        .setAutoCancel(true)
                        .build());
    }
}
```
Статический приёмник должен ловить то же действие:
```xml
<receiver android:name=".receivers.DownloadDoneReceiver" android:exported="false">
    <intent-filter>
        <action android:name="com.university.app.DOWNLOAD_DONE" />
    </intent-filter>
</receiver>
```
(замените им короткую запись выше).

```java
package com.university.app;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;

import com.university.app.services.LoaderService;

public class LoaderActivity extends AppCompatActivity {
    private TextView status;

    private final BroadcastReceiver uiReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            status.setText(intent.getStringExtra(LoaderService.EXTRA_RESULT));
            abortBroadcast(); // экран открыт — уведомление не нужно
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LinearLayout root = new LinearLayout(this);
        root.setOrientation(LinearLayout.VERTICAL);
        root.setPadding(32, 32, 32, 32);
        status = new TextView(this);
        status.setText("Готов к загрузке");
        Button start = new Button(this);
        start.setText("Скачать");
        start.setOnClickListener(v -> {
            status.setText("Идёт загрузка...");
            ContextCompat.startForegroundService(this, new Intent(this, LoaderService.class));
        });
        root.addView(status);
        root.addView(start);
        setContentView(root);
    }

    @Override
    protected void onStart() {
        super.onStart();
        IntentFilter filter = new IntentFilter(LoaderService.ACTION_DONE);
        filter.setPriority(100); // выше, чем у статического приёмника (по умолчанию 0)
        ContextCompat.registerReceiver(this, uiReceiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    protected void onStop() {
        super.onStop();
        unregisterReceiver(uiReceiver);
    }
}
```
Проверка: нажмите «Скачать»: сверху обновится текст. Нажмите «Скачать» и сразу сверните приложение: через 4 секунды придёт уведомление.

---

### 32. «Аварийный сброс» при низком заряде

Приёмник `ACTION_BATTERY_LOW` (динамический, регистрируется в `Application`) останавливает тяжёлую синхронизацию и через метод `call()` провайдера переводит данные в режим энергосбережения. `BATTERY_OKAY` возвращает всё обратно.

```xml
<provider android:name=".providers.DataProvider"
    android:authorities="com.university.app.data" android:exported="false" />
<service android:name=".services.SyncService" android:exported="false" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentValues;
import android.database.Cursor;
import android.database.MatrixCursor;
import android.net.Uri;
import android.os.Bundle;
import android.util.Log;

public class DataProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.data";
    public static final Uri BASE_URI = Uri.parse("content://" + AUTHORITY);
    public static final String METHOD_SET_POWER_SAVE = "set_power_save";
    public static final String KEY_ENABLED = "enabled";

    private volatile boolean powerSave = false;

    @Override
    public boolean onCreate() {
        return true;
    }

    /** Служебная команда для провайдера: ContentResolver.call(...). */
    @Override
    public Bundle call(String method, String arg, Bundle extras) {
        if (METHOD_SET_POWER_SAVE.equals(method) && extras != null) {
            powerSave = extras.getBoolean(KEY_ENABLED, false);
            Log.w("DataProvider", "Режим энергосбережения: " + powerSave);
            getContext().getContentResolver().notifyChange(BASE_URI, null);
            return Bundle.EMPTY;
        }
        return super.call(method, arg, extras);
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        MatrixCursor c = new MatrixCursor(new String[]{"_id", "value"});
        int rows = powerSave ? 20 : 200; // в режиме экономии отдаём урезанный набор
        for (int i = 0; i < rows; i++) {
            c.addRow(new Object[]{i, "Данные " + i});
        }
        c.setNotificationUri(getContext().getContentResolver(), uri);
        return c;
    }

    @Override public String getType(Uri uri) { return "vnd.android.cursor.dir/vnd.com.university.data"; }
    @Override public Uri insert(Uri uri, ContentValues values) { return null; }
    @Override public int delete(Uri uri, String s, String[] a) { return 0; }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { return 0; }
}
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

public class SyncService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d("SyncService", "Тяжёлая синхронизация запущена");
        return START_STICKY;
    }

    @Override
    public void onDestroy() {
        Log.d("SyncService", "Синхронизация остановлена");
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

```java
package com.university.app;

import android.app.Application;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Bundle;

import androidx.core.content.ContextCompat;

import com.university.app.providers.DataProvider;
import com.university.app.services.SyncService;

public class App extends Application {

    private final BroadcastReceiver batteryReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            boolean low = Intent.ACTION_BATTERY_LOW.equals(intent.getAction());
            if (low) {
                context.stopService(new Intent(context, SyncService.class)); // 1) гасим тяжёлое
            } else {
                // BATTERY_OKAY: из фона запускать сервис нельзя — перезапуск сделает WorkManager/экран
            }
            Bundle extras = new Bundle();
            extras.putBoolean(DataProvider.KEY_ENABLED, low);
            context.getContentResolver().call(DataProvider.BASE_URI, // 2) режим экономии в провайдере
                    DataProvider.METHOD_SET_POWER_SAVE, null, extras);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        IntentFilter filter = new IntentFilter();
        filter.addAction(Intent.ACTION_BATTERY_LOW);
        filter.addAction(Intent.ACTION_BATTERY_OKAY);
        ContextCompat.registerReceiver(this, batteryReceiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED);
    }
}
```
Выбор `call()` вместо отдельной «таблицы настроек» — осознанный: это штатный способ отправить провайдеру команду.

---

### 33. «Кэширующий прокси»: Provider → Service → `notifyChange()`

Если локальная БД пуста, `query()` возвращает пустой курсор (ничего не блокируя) и запускает сервис, который загружает данные, кладёт их в БД и уведомляет UI. Экран, подписанный на курсор, обновится сам.

```xml
<provider android:name=".providers.NewsProvider"
    android:authorities="com.university.app.news" android:exported="false" />
<service android:name=".services.NewsFetchService" android:exported="false" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentValues;
import android.content.Context;
import android.content.Intent;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.net.Uri;
import android.util.Log;

import com.university.app.services.NewsFetchService;

import java.util.concurrent.atomic.AtomicBoolean;

public class NewsProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.news";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/news");
    public static final AtomicBoolean FETCHING = new AtomicBoolean(false);

    private static class Db extends SQLiteOpenHelper {
        Db(Context c) {
            super(c, "news.db", null, 1);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE news (_id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT NOT NULL)");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int o, int n) {
            db.execSQL("DROP TABLE IF EXISTS news");
            onCreate(db);
        }
    }

    private Db helper;

    @Override
    public boolean onCreate() {
        helper = new Db(getContext());
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        Cursor c = helper.getReadableDatabase().query("news", projection, selection, selectionArgs,
                null, null, sortOrder);
        c.setNotificationUri(getContext().getContentResolver(), CONTENT_URI);
        if (c.getCount() == 0 && FETCHING.compareAndSet(false, true)) {
            try {
                getContext().startService(new Intent(getContext(), NewsFetchService.class));
            } catch (IllegalStateException e) { // приложение в фоне — запуск запрещён
                FETCHING.set(false);
                Log.w("NewsProvider", "Не удалось запустить загрузку из фона", e);
            }
        }
        return c;
    }

    @Override
    public int bulkInsert(Uri uri, ContentValues[] values) {
        SQLiteDatabase db = helper.getWritableDatabase();
        db.beginTransaction();
        try {
            for (ContentValues v : values) {
                db.insertOrThrow("news", null, v);
            }
            db.setTransactionSuccessful();
        } finally {
            db.endTransaction();
        }
        getContext().getContentResolver().notifyChange(CONTENT_URI, null); // UI обновится сам
        return values.length;
    }

    @Override public String getType(Uri uri) { return "vnd.android.cursor.dir/vnd.com.university.news"; }
    @Override public Uri insert(Uri uri, ContentValues values) { throw new UnsupportedOperationException(); }
    @Override public int delete(Uri uri, String s, String[] a) { throw new UnsupportedOperationException(); }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { throw new UnsupportedOperationException(); }
}
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.ContentValues;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

import com.university.app.providers.NewsProvider;

public class NewsFetchService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        new Thread(() -> {
            try {
                Thread.sleep(2000); // здесь настоящий сетевой запрос
                ContentValues[] rows = new ContentValues[10];
                for (int i = 0; i < rows.length; i++) {
                    rows[i] = new ContentValues();
                    rows[i].put("title", "Новость №" + (i + 1));
                }
                getContentResolver().bulkInsert(NewsProvider.CONTENT_URI, rows);
                Log.d("NewsFetch", "Кэш заполнен");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                NewsProvider.FETCHING.set(false);
                stopSelf(startId);
            }
        }, "news-fetch").start();
        return START_NOT_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Экран подписывается так же, как в задании 5.5 (`registerContentObserver(NewsProvider.CONTENT_URI, true, observer)`) и перечитывает данные в `onChange()`.

---

### 34. Управление плеером из `adb shell am broadcast`

Для безопасности приёмник регистрируется как экспортируемый **только в debug-сборке**, иначе любое приложение смогло бы управлять вашим плеером.

```java
package com.university.app.services;

import android.app.Service;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.ApplicationInfo;
import android.content.pm.ServiceInfo;
import android.media.AudioManager;
import android.media.MediaPlayer;
import android.os.IBinder;
import android.util.Log;

import androidx.core.app.ServiceCompat;
import androidx.core.content.ContextCompat;

import com.university.app.R;
import com.university.app.util.Notifs;

public class RemoteControlPlayerService extends Service {
    public static final String ACTION = "com.university.app.PLAYER_CONTROL";
    private MediaPlayer player;
    private AudioManager audio;

    private final BroadcastReceiver controlReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String cmd = intent.getStringExtra("command");
            if (cmd == null || player == null) return;
            switch (cmd) {
                case "play":
                    player.start();
                    break;
                case "stop":
                    if (player.isPlaying()) player.pause();
                    break;
                case "mute":
                    audio.adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_MUTE, 0);
                    break;
                case "unmute":
                    audio.adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_UNMUTE, 0);
                    break;
                default:
                    Log.w("RemoteControl", "Неизвестная команда: " + cmd);
            }
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        audio = getSystemService(AudioManager.class);
        player = MediaPlayer.create(this, R.raw.track1);
        boolean debuggable = (getApplicationInfo().flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
        ContextCompat.registerReceiver(this, controlReceiver, new IntentFilter(ACTION),
                debuggable ? ContextCompat.RECEIVER_EXPORTED : ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 334,
                Notifs.build(this, "Плеер", "Готов принимать команды"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK);
        return START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        unregisterReceiver(controlReceiver);
        if (player != null) player.release();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Манифест: `<service android:name=".services.RemoteControlPlayerService" android:exported="false" android:foregroundServiceType="mediaPlayback" />` + `FOREGROUND_SERVICE_MEDIA_PLAYBACK`.

Команды с компьютера (`-p` обязателен: он адресует бродкаст вашему приложению):
```
adb shell am broadcast -a com.university.app.PLAYER_CONTROL -p com.university.app --es command play
adb shell am broadcast -a com.university.app.PLAYER_CONTROL -p com.university.app --es command mute
adb shell am broadcast -a com.university.app.PLAYER_CONTROL -p com.university.app --es command stop
```

---

### 35. Двусторонняя связь Activity ↔ Bound Service (Callback/Listener)

Сервис сам «толкает» данные датчика освещённости в Activity. Подписку обязательно снимаем до `unbindService()`, иначе сервис будет держать ссылку на закрытую Activity (утечка).

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.hardware.Sensor;
import android.hardware.SensorEvent;
import android.hardware.SensorEventListener;
import android.hardware.SensorManager;
import android.os.Binder;
import android.os.IBinder;

import java.util.concurrent.CopyOnWriteArrayList;

public class LightSensorService extends Service implements SensorEventListener {

    public interface Listener {
        void onNewData(float lux);
    }

    private final CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();
    private final IBinder binder = new LocalBinder();
    private SensorManager sensorManager;

    public class LocalBinder extends Binder {
        public LightSensorService getService() {
            return LightSensorService.this;
        }
    }

    public void addListener(Listener l) {
        listeners.addIfAbsent(l);
    }

    public void removeListener(Listener l) {
        listeners.remove(l);
    }

    @Override
    public void onCreate() {
        super.onCreate();
        sensorManager = getSystemService(SensorManager.class);
        Sensor light = sensorManager.getDefaultSensor(Sensor.TYPE_LIGHT);
        if (light != null) {
            sensorManager.registerListener(this, light, SensorManager.SENSOR_DELAY_NORMAL);
        }
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        for (Listener l : listeners) { // колбэк приходит в главном потоке — можно обновлять UI
            l.onNewData(event.values[0]);
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) { }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    @Override
    public void onDestroy() {
        sensorManager.unregisterListener(this);
        listeners.clear();
        super.onDestroy();
    }
}
```

```java
package com.university.app;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.widget.TextView;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.services.LightSensorService;

import java.util.Locale;

public class LightActivity extends AppCompatActivity implements LightSensorService.Listener {
    private TextView view;
    private LightSensorService service;
    private boolean bound = false;

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName n, IBinder b) {
            service = ((LightSensorService.LocalBinder) b).getService();
            service.addListener(LightActivity.this); // подписались
            bound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName n) {
            service = null;
            bound = false;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        view = new TextView(this);
        view.setTextSize(24);
        view.setPadding(32, 32, 32, 32);
        view.setText("Ждём данные датчика...");
        setContentView(view);
    }

    @Override
    public void onNewData(float lux) {
        view.setText(String.format(Locale.getDefault(), "Освещённость: %.0f лк", lux));
    }

    @Override
    protected void onStart() {
        super.onStart();
        bindService(new Intent(this, LightSensorService.class), connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (bound) {
            service.removeListener(this); // сначала отписка
            unbindService(connection);    // потом отвязка
            bound = false;
        }
    }
}
```

---

### 36. Автобэкап базы в JSON по `ACTION_TIME_TICK`

`ACTION_TIME_TICK` приходит раз в минуту и только динамическому приёмнику. Тики могут пропускаться, пока устройство спит, поэтому считаем не количество тиков, а реальное время с прошлой копии. Читаем данные через `ContentResolver` из `NotesProvider` (часть 3), хранится последние 5 копий.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<service android:name=".services.BackupService" android:exported="false"
    android:foregroundServiceType="specialUse">
    <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="periodic_backup" />
</service>
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.ServiceInfo;
import android.database.Cursor;
import android.os.IBinder;
import android.util.Log;

import androidx.core.app.ServiceCompat;
import androidx.core.content.ContextCompat;

import com.university.app.providers.NotesProvider;
import com.university.app.util.Notifs;

import org.json.JSONArray;
import org.json.JSONException;
import org.json.JSONObject;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Arrays;
import java.util.Comparator;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BackupService extends Service {
    private static final String TAG = "BackupService";
    private static final long INTERVAL_MS = 60L * 60 * 1000; // не чаще раза в час
    private static final int KEEP = 5;

    private ExecutorService executor;
    private long lastBackup = 0;

    private final BroadcastReceiver tickReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            long now = System.currentTimeMillis();
            if (now - lastBackup >= INTERVAL_MS) {
                lastBackup = now;
                executor.execute(BackupService.this::doBackup);
            }
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        executor = Executors.newSingleThreadExecutor();
        // TIME_TICK не принимается через манифест — только динамически
        ContextCompat.registerReceiver(this, tickReceiver, new IntentFilter(Intent.ACTION_TIME_TICK),
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 336,
                Notifs.build(this, "Резервное копирование", "Копия раз в час"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE);
        return START_STICKY;
    }

    private void doBackup() {
        JSONArray array = new JSONArray();
        try (Cursor c = getContentResolver().query(NotesProvider.CONTENT_URI,
                new String[]{"_id", "title", "body", "category"}, null, null, "_id ASC")) {
            while (c != null && c.moveToNext()) {
                JSONObject o = new JSONObject();
                o.put("id", c.getLong(0));
                o.put("title", c.getString(1));
                o.put("body", c.getString(2));
                o.put("category", c.getString(3));
                array.put(o);
            }
        } catch (JSONException e) {
            Log.e(TAG, "Ошибка формирования JSON", e);
            return;
        }

        File dir = new File(getFilesDir(), "backups");
        if (!dir.exists() && !dir.mkdirs()) {
            return;
        }
        File out = new File(dir, "notes_" + System.currentTimeMillis() + ".json");
        try (FileOutputStream fos = new FileOutputStream(out)) {
            fos.write(array.toString().getBytes(StandardCharsets.UTF_8));
            Log.d(TAG, "Копия: " + out.getName() + ", записей: " + array.length());
        } catch (IOException e) {
            Log.e(TAG, "Не удалось записать копию", e);
        }
        pruneOld(dir);
    }

    private void pruneOld(File dir) {
        File[] files = dir.listFiles();
        if (files == null || files.length <= KEEP) return;
        Arrays.sort(files, Comparator.comparingLong(File::lastModified));
        for (int i = 0; i < files.length - KEEP; i++) {
            files[i].delete();
        }
    }

    @Override
    public void onDestroy() {
        unregisterReceiver(tickReceiver);
        executor.shutdownNow();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Для проверки временно поставьте `INTERVAL_MS = 60_000`. В реальном проекте такую задачу правильно делать через `PeriodicWorkRequest` (WorkManager): надёжнее и экономнее по батарее.

---

### 37. Белый список вызывающих пакетов

Определяем вызывающего по UID, который выставляет сама система (подделать нельзя), а не по строке из интента. Дополнительно сверяем подпись: имя пакета можно «занять» в чужом приложении, подпись нет. Использовать `getPackagesForUid()`, а не `getNameForUid()`: для приложений с общим UID `getNameForUid()` возвращает не имя пакета.

```java
package com.university.app.security;

import android.content.Context;
import android.content.pm.PackageManager;
import android.os.Binder;
import android.os.Process;

import java.util.Arrays;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

public final class CallerGuard {
    private static final Set<String> ALLOWED = Collections.unmodifiableSet(
            new HashSet<>(Arrays.asList("com.company.partner", "com.company.analytics")));

    private CallerGuard() { }

    /** Вызывать в самом начале Binder-метода (query, onTransact), в потоке Binder. */
    public static void enforce(Context ctx) {
        int callingUid = Binder.getCallingUid();
        if (callingUid == Process.myUid()) {
            return; // вызов из нашего же приложения
        }
        PackageManager pm = ctx.getPackageManager();
        String[] packages = pm.getPackagesForUid(callingUid);
        if (packages != null) {
            for (String pkg : packages) {
                if (ALLOWED.contains(pkg)
                        && pm.checkSignatures(Process.myUid(), callingUid) == PackageManager.SIGNATURE_MATCH) {
                    return;
                }
            }
        }
        throw new SecurityException("Вызов запрещён для uid " + callingUid);
    }
}
```

Использование в провайдере:
```java
@Override
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
    CallerGuard.enforce(getContext()); // первая строка
    // ... дальше обычная логика
}
```

Использование в Bound-сервисе с «голым» `Binder`:
```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;
import android.os.Parcel;
import android.os.RemoteException;

import com.university.app.security.CallerGuard;

public class GuardedService extends Service {
    private final IBinder binder = new Binder() {
        @Override
        protected boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            CallerGuard.enforce(GuardedService.this); // проверка при каждом обращении
            return super.onTransact(code, data, reply, flags);
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```
Важно: `Binder.getCallingUid()` корректен только в потоке, который принял вызов. Если сначала передать работу в свой `Executor`, он вернёт UID вашего собственного процесса, и проверка станет бесполезной. Для Android 11+ вызывающее приложение должно быть видимо вашему (`<queries>`), иначе `getPackagesForUid` может вернуть `null`; в этом случае доступ **запрещается** (так и написано в коде).

---

### 38. Потоковая передача из сервиса в провайдер через `ParcelFileDescriptor.createPipe()`

Сервис — источник потока, провайдер — «труба» для клиентов. Промежуточный файл на диске не создаётся. Сервис и провайдер живут в одном процессе, поэтому общаются через простой синглтон. Если бы сервис был в другом процессе, сервис передавал бы провайдеру записывающий конец канала через Binder (`ParcelFileDescriptor` умеет передаваться через IPC).

```xml
<provider android:name=".providers.StreamProvider"
    android:authorities="com.university.app.stream" android:exported="false"
    android:grantUriPermissions="true" />
<service android:name=".services.DataProducerService" android:exported="false" />
```

```java
package com.university.app.stream;

import java.io.IOException;
import java.io.OutputStream;

public final class StreamHub {
    public interface Producer {
        void writeTo(OutputStream out) throws IOException;
    }

    public static volatile Producer producer;

    private StreamHub() { }
}
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;

import com.university.app.stream.StreamHub;

import java.io.IOException;
import java.io.OutputStream;
import java.nio.charset.StandardCharsets;

public class DataProducerService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        StreamHub.producer = (OutputStream out) -> {
            for (int i = 1; i <= 100_000; i++) { // данные рождаются «на лету»
                out.write(("Строка данных №" + i + "\n").getBytes(StandardCharsets.UTF_8));
            }
        };
        return START_STICKY;
    }

    @Override
    public void onDestroy() {
        StreamHub.producer = null;
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentValues;
import android.database.Cursor;
import android.net.Uri;
import android.os.ParcelFileDescriptor;

import com.university.app.stream.StreamHub;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.OutputStream;

public class StreamProvider extends ContentProvider {

    @Override
    public boolean onCreate() {
        return true;
    }

    @Override
    public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
        if (!"r".equals(mode)) {
            throw new SecurityException("Только чтение");
        }
        final StreamHub.Producer producer = StreamHub.producer;
        if (producer == null) {
            throw new FileNotFoundException("Сервис-источник не запущен");
        }
        try {
            ParcelFileDescriptor[] pipe = ParcelFileDescriptor.createPipe(); // [0] — чтение, [1] — запись
            final ParcelFileDescriptor writeSide = pipe[1];
            new Thread(() -> {
                try (OutputStream out = new ParcelFileDescriptor.AutoCloseOutputStream(writeSide)) {
                    producer.writeTo(out);
                } catch (IOException e) {
                    // клиент закрыл чтение раньше — обычная ситуация
                }
            }, "stream-writer").start();
            return pipe[0]; // клиент читает отсюда
        } catch (IOException e) {
            throw new FileNotFoundException("Не удалось создать канал: " + e.getMessage());
        }
    }

    @Override public String getType(Uri uri) { return "text/plain"; }
    @Override public Cursor query(Uri u, String[] p, String s, String[] a, String o) { return null; }
    @Override public Uri insert(Uri uri, ContentValues values) { return null; }
    @Override public int delete(Uri uri, String s, String[] a) { return 0; }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { return 0; }
}
```

Клиент (в фоновом потоке):
```java
Uri uri = Uri.parse("content://com.university.app.stream/data");
try (java.io.BufferedReader r = new java.io.BufferedReader(
        new java.io.InputStreamReader(getContentResolver().openInputStream(uri)))) {
    String line;
    int count = 0;
    while ((line = r.readLine()) != null && count++ < 5) {
        android.util.Log.d("Stream", line);
    }
} catch (Exception e) {
    android.util.Log.e("Stream", "Ошибка чтения", e);
}
```
Готовый вспомогательный вариант тех же действий — `ContentProvider.openPipeHelper()` с `PipeDataWriter` (пример в задании 28): внутри него то же самое `createPipe()`.

---

### 39. Имитация системного будильника: `AlarmManager` + Receiver + Foreground Service

Цепочка: `setAlarmClock()` → `AlarmReceiver` → foreground-сервис с мелодией → уведомление `fullScreenIntent` открывает экран будильника поверх блокировки. Запуск foreground-сервиса из приёмника точного будильника системой разрешён.

```xml
<uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<receiver android:name=".alarm.AlarmReceiver" android:exported="false" />
<service android:name=".alarm.AlarmService" android:exported="false"
    android:foregroundServiceType="mediaPlayback" />
<activity android:name=".alarm.AlarmRingActivity" android:exported="false"
    android:showWhenLocked="true" android:turnScreenOn="true"
    android:launchMode="singleInstance" />
```
Замечание: на Android 14+ `USE_FULL_SCREEN_INTENT` без особого статуса «приложение-будильник» может требовать, чтобы пользователь вручную разрешил полноэкранные уведомления (`NotificationManager.canUseFullScreenIntent()`). Если не разрешено, система покажет обычное уведомление с кнопкой.

```java
package com.university.app.alarm;

import android.app.AlarmManager;
import android.app.PendingIntent;
import android.content.Context;
import android.content.Intent;

public final class AlarmScheduler {
    private AlarmScheduler() { }

    public static void schedule(Context ctx, long triggerAtMillis) {
        PendingIntent fire = PendingIntent.getBroadcast(ctx, 1,
                new Intent(ctx, AlarmReceiver.class),
                PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_UPDATE_CURRENT);
        PendingIntent show = PendingIntent.getActivity(ctx, 2,
                new Intent(ctx, AlarmRingActivity.class), PendingIntent.FLAG_IMMUTABLE);
        // setAlarmClock: точный, срабатывает даже в Doze, отдельного разрешения не требует
        ctx.getSystemService(AlarmManager.class)
                .setAlarmClock(new AlarmManager.AlarmClockInfo(triggerAtMillis, show), fire);
    }
}
```

```java
package com.university.app.alarm;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;

import androidx.core.content.ContextCompat;

public class AlarmReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        ContextCompat.startForegroundService(context, new Intent(context, AlarmService.class));
    }
}
```

```java
package com.university.app.alarm;

import android.app.Notification;
import android.app.PendingIntent;
import android.app.Service;
import android.content.Intent;
import android.content.pm.ServiceInfo;
import android.media.AudioAttributes;
import android.media.Ringtone;
import android.media.RingtoneManager;
import android.net.Uri;
import android.os.Build;
import android.os.IBinder;

import androidx.core.app.NotificationCompat;
import androidx.core.app.ServiceCompat;

import com.university.app.util.Notifs;

public class AlarmService extends Service {
    public static final String ACTION_STOP = "com.university.app.ALARM_STOP";
    private Ringtone ringtone;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent != null && ACTION_STOP.equals(intent.getAction())) {
            stopSelf();
            return START_NOT_STICKY;
        }
        Notifs.ensureChannel(this);
        PendingIntent full = PendingIntent.getActivity(this, 3,
                new Intent(this, AlarmRingActivity.class).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK),
                PendingIntent.FLAG_IMMUTABLE);
        PendingIntent stop = PendingIntent.getService(this, 4,
                new Intent(this, AlarmService.class).setAction(ACTION_STOP),
                PendingIntent.FLAG_IMMUTABLE);

        Notification n = new NotificationCompat.Builder(this, Notifs.CHANNEL)
                .setSmallIcon(android.R.drawable.ic_lock_idle_alarm)
                .setContentTitle("Будильник")
                .setContentText("Пора вставать!")
                .setCategory(NotificationCompat.CATEGORY_ALARM)
                .setPriority(NotificationCompat.PRIORITY_MAX)
                .setFullScreenIntent(full, true) // будит экран
                .addAction(android.R.drawable.ic_menu_close_clear_cancel, "Стоп", stop)
                .setOngoing(true)
                .build();
        ServiceCompat.startForeground(this, 339, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK);

        Uri sound = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM);
        ringtone = RingtoneManager.getRingtone(this, sound);
        ringtone.setAudioAttributes(new AudioAttributes.Builder()
                .setUsage(AudioAttributes.USAGE_ALARM)
                .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION).build());
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            ringtone.setLooping(true);
        }
        ringtone.play();
        return START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        if (ringtone != null) ringtone.stop();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

```java
package com.university.app.alarm;

import android.content.Intent;
import android.os.Bundle;
import android.widget.Button;

import androidx.appcompat.app.AppCompatActivity;

public class AlarmRingActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button stop = new Button(this);
        stop.setText("ВЫКЛЮЧИТЬ БУДИЛЬНИК");
        stop.setTextSize(24);
        stop.setOnClickListener(v -> {
            startService(new Intent(this, AlarmService.class).setAction(AlarmService.ACTION_STOP));
            finish();
        });
        setContentView(stop);
    }
}
```
Установка: `AlarmScheduler.schedule(this, System.currentTimeMillis() + 30_000);` (через 30 секунд).

---

### 40. Динамическое включение/выключение приёмника: `setComponentEnabledSetting()`

В манифесте приёмник объявлен выключенным. Состояние сохраняется между перезапусками приложения (оно хранится в системе).

```xml
<receiver android:name=".receivers.OptionalReceiver" android:enabled="false" android:exported="false">
    <intent-filter>
        <action android:name="com.university.app.OPTIONAL_EVENT" />
    </intent-filter>
</receiver>
```

```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class OptionalReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.d("OptionalReceiver", "Приёмник включён и получил событие");
    }
}
```

```java
package com.university.app;

import android.content.ComponentName;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.widget.CompoundButton;
import android.widget.Switch;

import androidx.appcompat.app.AppCompatActivity;

import com.university.app.receivers.OptionalReceiver;

public class ToggleReceiverActivity extends AppCompatActivity {

    private boolean isEnabled(ComponentName component) {
        int state = getPackageManager().getComponentEnabledSetting(component);
        // DEFAULT = «как в манифесте»; у нас в манифесте enabled="false"
        return state == PackageManager.COMPONENT_ENABLED_STATE_ENABLED;
    }

    private void setEnabled(ComponentName component, boolean enable) {
        getPackageManager().setComponentEnabledSetting(component,
                enable ? PackageManager.COMPONENT_ENABLED_STATE_ENABLED
                        : PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                PackageManager.DONT_KILL_APP); // иначе приложение перезапустится
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        final ComponentName component = new ComponentName(this, OptionalReceiver.class);

        Switch toggle = new Switch(this);
        toggle.setText("Приёмник включён");
        toggle.setPadding(32, 32, 32, 32);
        toggle.setChecked(isEnabled(component));
        toggle.setOnCheckedChangeListener((CompoundButton b, boolean checked) -> setEnabled(component, checked));
        setContentView(toggle);
    }
}
```
Проверка (включите переключатель, затем):
```
adb shell am broadcast -a com.university.app.OPTIONAL_EVENT -p com.university.app
```
При включённом переключателе в логе появится строка, при выключенном нет.

---

## Блок 5. Безопасность, оптимизация, отказоустойчивость

### 41. Защита от replay-атак: таймстемп + одноразовая соль (nonce) + HMAC

Приёмник принимает сообщение, только если (1) подпись HMAC верна, (2) время не старше 30 секунд, (3) такой nonce ещё не встречался. Подмена времени или nonce ломает подпись, а повторная отправка старого перехваченного интента отвергается.

```java
package com.university.app.security;

import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.util.Base64;

import java.nio.charset.StandardCharsets;
import java.security.GeneralSecurityException;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.LinkedHashMap;
import java.util.Map;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

public final class SecureBroadcast {
    private static final long MAX_AGE_MS = 30_000;
    private static final String X_PAYLOAD = "payload", X_TS = "ts", X_NONCE = "nonce", X_MAC = "mac";

    // Запомненные nonce; не даём коллекции расти бесконечно
    private static final Map<String, Long> SEEN = new LinkedHashMap<String, Long>(64, 0.75f, false) {
        @Override
        protected boolean removeEldestEntry(Map.Entry<String, Long> eldest) {
            return size() > 500;
        }
    };

    private SecureBroadcast() { }

    public static Intent sign(Context ctx, String action, String payload) {
        long ts = System.currentTimeMillis();
        byte[] nonceBytes = new byte[16];
        new SecureRandom().nextBytes(nonceBytes);
        String nonce = Base64.encodeToString(nonceBytes, Base64.NO_WRAP);
        Intent i = new Intent(action);
        i.setPackage(ctx.getPackageName());
        i.putExtra(X_PAYLOAD, payload);
        i.putExtra(X_TS, ts);
        i.putExtra(X_NONCE, nonce);
        i.putExtra(X_MAC, mac(ctx, action, payload, ts, nonce));
        return i;
    }

    public static boolean verify(Context ctx, Intent i) {
        String action = i.getAction();
        String payload = i.getStringExtra(X_PAYLOAD);
        String nonce = i.getStringExtra(X_NONCE);
        String given = i.getStringExtra(X_MAC);
        long ts = i.getLongExtra(X_TS, 0);
        if (action == null || payload == null || nonce == null || given == null || ts == 0) {
            return false;
        }
        if (Math.abs(System.currentTimeMillis() - ts) > MAX_AGE_MS) {
            return false; // слишком старое (или из будущего)
        }
        // Сравнение за постоянное время, чтобы не подсказывать побайтово
        if (!MessageDigest.isEqual(mac(ctx, action, payload, ts, nonce).getBytes(StandardCharsets.UTF_8),
                given.getBytes(StandardCharsets.UTF_8))) {
            return false;
        }
        synchronized (SEEN) {
            if (SEEN.containsKey(nonce)) {
                return false; // повтор!
            }
            SEEN.put(nonce, ts);
        }
        return true;
    }

    private static String mac(Context ctx, String action, String payload, long ts, String nonce) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret(ctx), "HmacSHA256"));
            byte[] out = mac.doFinal((action + "|" + payload + "|" + ts + "|" + nonce)
                    .getBytes(StandardCharsets.UTF_8));
            return Base64.encodeToString(out, Base64.NO_WRAP);
        } catch (GeneralSecurityException e) {
            throw new IllegalStateException(e);
        }
    }

    // Общий секрет, создаётся один раз и хранится в приватном хранилище приложения
    private static synchronized byte[] secret(Context ctx) {
        SharedPreferences p = ctx.getSharedPreferences("secure_bc", Context.MODE_PRIVATE);
        String s = p.getString("key", null);
        if (s == null) {
            byte[] k = new byte[32];
            new SecureRandom().nextBytes(k);
            s = Base64.encodeToString(k, Base64.NO_WRAP);
            p.edit().putString("key", s).apply();
        }
        return Base64.decode(s, Base64.NO_WRAP);
    }
}
```

Приёмник и отправка:
```java
package com.university.app.receivers;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

import com.university.app.security.SecureBroadcast;

public class ReplayProtectedReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (!SecureBroadcast.verify(context, intent)) {
            Log.w("ReplayGuard", "ОТКЛОНЕНО: неверная подпись, устарело или повтор");
            return;
        }
        Log.d("ReplayGuard", "Принято: " + intent.getStringExtra("payload"));
    }
}
```
```java
// Отправка (дважды один и тот же интент — второй раз будет отклонён):
Intent msg = SecureBroadcast.sign(this, "com.university.app.PROTECTED_EVENT", "перевод 100 руб.");
sendBroadcast(msg);
sendBroadcast(msg); // имитация replay → в логе «ОТКЛОНЕНО»
```
Нюансы: список nonce хранится в памяти, после перезапуска процесса он пуст (повтор в пределах 30 секунд теоретически пройдёт; при необходимости храните nonce в БД). Если сообщения идут между **двумя** приложениями, общий секрет нужно передать защищённо, а не генерировать у каждого свой.

---

### 42. Защита от Path Traversal в `openFile()`

Главное: сравниваем **канонический** путь (после раскрытия `..` и символических ссылок) с базовой папкой.

```xml
<provider android:name=".providers.DocsProvider"
    android:authorities="com.university.app.docs" android:exported="true"
    android:readPermission="com.university.permission.READ_DOCS" />
<permission android:name="com.university.permission.READ_DOCS" android:protectionLevel="signature" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentValues;
import android.database.Cursor;
import android.net.Uri;
import android.os.ParcelFileDescriptor;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;

public class DocsProvider extends ContentProvider {

    private File baseDir;

    @Override
    public boolean onCreate() {
        baseDir = new File(getContext().getFilesDir(), "docs");
        return true;
    }

    @Override
    public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
        if (!"r".equals(mode)) {
            throw new SecurityException("Только чтение");
        }
        String relative = uri.getPath(); // уже декодирован: %2e%2e станет ".."
        if (relative == null || relative.contains("\u0000")) {
            throw new SecurityException("Недопустимый путь");
        }
        try {
            File base = baseDir.getCanonicalFile();
            File target = new File(base, relative).getCanonicalFile(); // раскрывает "..", "." и symlink
            // Обязательно с разделителем: иначе /docs_evil пройдёт проверку префикса "/docs"
            if (!target.getPath().startsWith(base.getPath() + File.separator)) {
                throw new SecurityException("Выход за пределы каталога: " + relative);
            }
            if (!target.isFile()) {
                throw new FileNotFoundException("Нет файла: " + relative);
            }
            return ParcelFileDescriptor.open(target, ParcelFileDescriptor.MODE_READ_ONLY);
        } catch (IOException e) {
            throw new FileNotFoundException("Ошибка пути");
        }
    }

    @Override public String getType(Uri uri) { return "application/octet-stream"; }
    @Override public Cursor query(Uri u, String[] p, String s, String[] a, String o) { return null; }
    @Override public Uri insert(Uri uri, ContentValues values) { return null; }
    @Override public int delete(Uri uri, String s, String[] a) { return 0; }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { return 0; }
}
```

Проверка:
```java
// Нормально:
getContentResolver().openInputStream(Uri.parse("content://com.university.app.docs/report.txt"));
// Атака (должна упасть с SecurityException):
getContentResolver().openInputStream(Uri.parse("content://com.university.app.docs/..%2Fdatabases%2Fnotes.db"));
```
(Предварительно положите `report.txt` в `files/docs/`.)

---

### 43. Раздельные `readPermission` и `writePermission`

```xml
<!-- Объявление разрешений (в приложении-владельце) -->
<permission android:name="com.university.app.permission.READ_DATA"
    android:protectionLevel="signature" />
<permission android:name="com.university.app.permission.WRITE_DATA"
    android:protectionLevel="signature" />
<permission android:name="com.university.app.permission.READ_PUBLIC"
    android:protectionLevel="normal" />

<provider
    android:name=".providers.NotesProvider"
    android:authorities="com.university.app.provider.notes"
    android:exported="true"
    android:readPermission="com.university.app.permission.READ_DATA"
    android:writePermission="com.university.app.permission.WRITE_DATA">

    <!-- Отдельный путь /notes/public читается с более мягким разрешением -->
    <path-permission
        android:pathPrefix="/notes/public"
        android:readPermission="com.university.app.permission.READ_PUBLIC" />
</provider>
```

Приложение-клиент объявляет только то, что ему нужно (например, только чтение):
```xml
<uses-permission android:name="com.university.app.permission.READ_DATA" />
<!-- WRITE_DATA не запрашиваем: запись будет запрещена -->
```

Как это работает: `query()` проверяется по `readPermission`; `insert()`/`update()`/`delete()` по `writePermission`. Без права система бросает `SecurityException` ещё до вызова кода провайдера.

Демонстрация на клиенте:
```java
package com.university.app;

import android.content.ContentResolver;
import android.content.ContentValues;
import android.database.Cursor;
import android.net.Uri;
import android.util.Log;

public class PermissionDemo {
    private static final Uri URI = Uri.parse("content://com.university.app.provider.notes/notes");

    public static void run(ContentResolver resolver) {
        try (Cursor c = resolver.query(URI, null, null, null, null)) {
            Log.d("PermDemo", "Чтение разрешено, строк: " + (c == null ? 0 : c.getCount()));
        } catch (SecurityException e) {
            Log.w("PermDemo", "Чтение запрещено: " + e.getMessage());
        }
        try {
            ContentValues v = new ContentValues();
            v.put("title", "Тест");
            resolver.insert(URI, v);
            Log.d("PermDemo", "Запись разрешена");
        } catch (SecurityException e) {
            Log.w("PermDemo", "Запись запрещена (ожидаемо): " + e.getMessage());
        }
    }
}
```
Ещё одна практика: если провайдер нужен только внутри приложения, ставьте `exported="false"`, тогда разрешения вообще не нужны.

---

### 44. Утечки памяти в сервисе: статические ссылки на Activity

**Плохо** (утечка):
```java
package com.university.app.services;

import android.app.Activity;
import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.widget.TextView;

public class LeakyService extends Service {
    // ОШИБКА №1: статическое поле живёт, пока жив процесс; Activity никогда не освободится
    public static Activity sActivity;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        new Thread(() -> {
            while (true) {
                if (sActivity != null) {
                    TextView t = new TextView(sActivity); // держит контекст Activity
                }
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    return;
                }
            }
        }).start(); // ОШИБКА №2: поток не останавливается в onDestroy
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
// В Activity: LeakyService.sActivity = this;  и нигде не обнуляется
```

**Хорошо** (исправлено):
```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;

import java.util.concurrent.CopyOnWriteArrayList;

public class FixedService extends Service {
    public interface Listener {
        void onTick(int n);
    }

    // Слушатели регистрирует и снимает сам клиент; сервис не знает про Activity как таковую
    private final CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();
    private final Handler handler = new Handler(Looper.getMainLooper());
    private final IBinder binder = new LocalBinder();
    private int ticks = 0;

    private final Runnable ticker = new Runnable() {
        @Override
        public void run() {
            for (Listener l : listeners) {
                l.onTick(++ticks);
            }
            handler.postDelayed(this, 1000);
        }
    };

    public class LocalBinder extends Binder {
        public void addListener(Listener l) {
            listeners.addIfAbsent(l);
        }

        public void removeListener(Listener l) {
            listeners.remove(l);
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        handler.post(ticker);
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    @Override
    public void onDestroy() {
        handler.removeCallbacks(ticker); // останавливаем «хвосты»
        listeners.clear();
        super.onDestroy();
    }
}
```
Правила: нет статических ссылок на `Activity`/`View`/`Context` экрана; в сервисе работаем с `getApplicationContext()`; клиент снимает подписку в `onStop()`; в `onDestroy()` останавливаем потоки и `Handler`.

**Как найти утечку (профилирование):**
1. Подключите LeakCanary: `debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.14'`. Закройте экран, и через несколько секунд придёт уведомление с цепочкой ссылок вида `LeakyService.sActivity → MainActivity`.
2. Вручную: Android Studio → View → Tool Windows → Profiler → Memory → «Capture heap dump». Найдите класс вашей Activity: после закрытия экрана их должно быть 0, а если остался экземпляр (включите фильтр «Activity/Fragment Leaks»), в «References» будет видно, что его держит.
3. Быстрая проверка: поверните экран 5 раз. Число живых экземпляров Activity в heap dump не должно расти.

---

### 45. Doze Mode: корректный `WakeLock` и реакция на сон

В Doze система **игнорирует wake lock'и** и приостанавливает сеть, поэтому «удержать устройство бодрствующим» нельзя. Правильная стратегия: брать `WakeLock` только на короткую реальную работу и обязательно с таймаутом, а при входе в Doze приостанавливать сетевую работу и возобновлять её при выходе.

```xml
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<service android:name=".services.DozeAwareService" android:exported="false"
    android:foregroundServiceType="dataSync" />
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.ServiceInfo;
import android.os.IBinder;
import android.os.PowerManager;
import android.util.Log;

import androidx.core.app.ServiceCompat;
import androidx.core.content.ContextCompat;

import com.university.app.util.Notifs;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicBoolean;

public class DozeAwareService extends Service {
    private static final String TAG = "DozeAware";
    private static final long WAKELOCK_TIMEOUT_MS = 2 * 60 * 1000L; // обязательный таймаут

    private final AtomicBoolean deviceIdle = new AtomicBoolean(false);
    private ExecutorService executor;
    private PowerManager powerManager;

    private final BroadcastReceiver idleReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            boolean idle = powerManager.isDeviceIdleMode();
            deviceIdle.set(idle);
            Log.w(TAG, idle ? "Вошли в Doze: приостанавливаем сетевую работу"
                    : "Вышли из Doze: возобновляем");
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        powerManager = getSystemService(PowerManager.class);
        executor = Executors.newSingleThreadExecutor();
        deviceIdle.set(powerManager.isDeviceIdleMode());
        ContextCompat.registerReceiver(this, idleReceiver,
                new IntentFilter(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED),
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 345,
                Notifs.build(this, "Синхронизация", "Работаю с учётом Doze"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC);
        executor.execute(this::loop);
        return START_STICKY;
    }

    private void loop() {
        while (!Thread.currentThread().isInterrupted()) {
            if (deviceIdle.get()) {
                sleep(5000); // в Doze ничего не делаем, ждём окно обслуживания
                continue;
            }
            PowerManager.WakeLock wl = powerManager.newWakeLock(
                    PowerManager.PARTIAL_WAKE_LOCK, "university:sync");
            wl.acquire(WAKELOCK_TIMEOUT_MS); // только на время одной порции работы
            try {
                Log.d(TAG, "Порция работы...");
                sleep(2000); // сетевой запрос/обработка
            } finally {
                if (wl.isHeld()) {
                    wl.release(); // освобождаем сразу, не ждём таймаута
                }
            }
            sleep(10_000);
        }
    }

    private void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public void onDestroy() {
        unregisterReceiver(idleReceiver);
        executor.shutdownNow();
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Для отложенной работы, которая должна пережить Doze, используйте `WorkManager` или `AlarmManager.setAndAllowWhileIdle()`. Проверка Doze на эмуляторе:
```
adb shell dumpsys deviceidle force-idle
adb shell dumpsys deviceidle unforce
```

---

### 46. Защита сервиса: `exported="false"` и Intent Injection

`exported="false"` закрывает сервис от **других приложений**, но не защищает от ошибок внутри вашего кода: если сервис слепо доверяет содержимому полученного `Intent` (например, запускает вложенный `Intent` из extras), ваша же Activity или чужой компонент через уязвимость может заставить его сделать лишнее. Поэтому дополнительно валидируем всё.

```xml
<service android:name=".services.SafeCommandService" android:exported="false" />
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

import java.util.Arrays;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

public class SafeCommandService extends Service {
    private static final String TAG = "SafeCommand";
    public static final String ACTION_SYNC = "com.university.app.cmd.SYNC";
    public static final String ACTION_CLEAR = "com.university.app.cmd.CLEAR";
    private static final Set<String> ALLOWED_ACTIONS = Collections.unmodifiableSet(
            new HashSet<>(Arrays.asList(ACTION_SYNC, ACTION_CLEAR)));
    private static final int MAX_ITEMS = 1000;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        try {
            if (intent == null || !ALLOWED_ACTIONS.contains(intent.getAction())) {
                Log.w(TAG, "Неизвестная команда отклонена");
                return stopAndReturn(startId);
            }
            // Читаем только ожидаемые типы; лишние/подменённые extras игнорируем
            int items = intent.getIntExtra("items", 0);
            if (items < 0 || items > MAX_ITEMS) {
                Log.w(TAG, "Недопустимый параметр items=" + items);
                return stopAndReturn(startId);
            }
            // НИКОГДА: startActivity((Intent) intent.getParcelableExtra("next")) — это и есть Intent Injection
            Log.d(TAG, "Выполняю " + intent.getAction() + ", items=" + items);
        } catch (RuntimeException e) {
            // BadParcelableException/ClassCastException при «ядовитых» extras не должны ронять процесс
            Log.e(TAG, "Некорректный Intent", e);
        }
        return stopAndReturn(startId);
    }

    private int stopAndReturn(int startId) {
        stopSelf(startId);
        return START_NOT_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Сопутствующие правила: `PendingIntent` создавайте с `FLAG_IMMUTABLE`; не принимайте от внешних источников `Intent` для последующего запуска; для Activity/Service/Receiver, которым не нужен внешний вызов, всегда пишите `exported="false"` явно.

---

### 47. Отказоустойчивость при повреждении SQLite

Все операции проходят через `execute()`: при `SQLiteDatabaseCorruptException` файл БД пересоздаётся, операция повторяется один раз, UI получает уведомление об изменении. Данные повреждённой базы теряются, поэтому в реальном проекте их потом восстанавливают с сервера или из резервной копии.

```xml
<provider android:name=".providers.ResilientProvider"
    android:authorities="com.university.app.resilient" android:exported="false" />
```

```java
package com.university.app.providers;

import android.content.ContentProvider;
import android.content.ContentUris;
import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteDatabaseCorruptException;
import android.database.sqlite.SQLiteOpenHelper;
import android.net.Uri;
import android.util.Log;

public class ResilientProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.resilient";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/items");
    private static final String DB_NAME = "resilient.db";
    private static final String TAG = "ResilientProvider";

    private interface DbOp<T> {
        T run(SQLiteDatabase db);
    }

    private static class Db extends SQLiteOpenHelper {
        Db(Context c) {
            super(c, DB_NAME, null, 1);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE items (_id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL)");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int o, int n) {
            db.execSQL("DROP TABLE IF EXISTS items");
            onCreate(db);
        }
    }

    private volatile Db helper;

    @Override
    public boolean onCreate() {
        helper = new Db(getContext());
        return true;
    }

    /** Выполняет операцию; при повреждении БД восстанавливает файл и повторяет один раз. */
    private <T> T execute(DbOp<T> op) {
        try {
            return op.run(helper.getWritableDatabase());
        } catch (SQLiteDatabaseCorruptException e) {
            Log.e(TAG, "БД повреждена — пересоздаём", e);
            recover();
            getContext().getContentResolver().notifyChange(CONTENT_URI, null);
            return op.run(helper.getWritableDatabase()); // вторая попытка на чистой базе
        }
    }

    private synchronized void recover() {
        helper.close();
        getContext().deleteDatabase(DB_NAME); // удаляет файл БД и сопутствующие -journal/-wal
        helper = new Db(getContext());
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        Cursor c = execute(db -> db.query("items", projection, selection, selectionArgs,
                null, null, sortOrder));
        c.setNotificationUri(getContext().getContentResolver(), CONTENT_URI);
        return c;
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        long id = execute(db -> db.insertOrThrow("items", null, values));
        getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return ContentUris.withAppendedId(CONTENT_URI, id);
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] args) {
        int n = execute(db -> db.update("items", values, selection, args));
        if (n > 0) getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return n;
    }

    @Override
    public int delete(Uri uri, String selection, String[] args) {
        int n = execute(db -> db.delete("items", selection, args));
        if (n > 0) getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return n;
    }

    @Override
    public String getType(Uri uri) {
        return "vnd.android.cursor.dir/vnd.com.university.item";
    }
}
```
Как проверить: вставьте запись, затем (на эмуляторе, debug-сборка) испортите файл: `adb shell run-as com.university.app sh -c "echo garbage > databases/resilient.db"`, перезапустите приложение и обратитесь к провайдеру: в логе появится «БД повреждена — пересоздаём», а приложение не упадёт. Замечание: если база повреждена уже на этапе открытия, `SQLiteOpenHelper` по умолчанию сам удаляет файл (`DefaultDatabaseErrorHandler`); код выше закрывает случай, когда повреждение обнаруживается позже, во время работы.

---

### 48. Снижение частоты акселерометра в 10 раз при выключенном экране

Датчики с непрерывной передачей данных в фоне на Android 9+ работают только у foreground-сервиса, поэтому сервис — foreground. 50 Гц при включённом экране (период 20 000 мкс) и 5 Гц при выключенном (200 000 мкс): ровно в 10 раз реже.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
<service android:name=".services.AccelService" android:exported="false"
    android:foregroundServiceType="specialUse">
    <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="motion_logging" />
</service>
```

```java
package com.university.app.services;

import android.app.Service;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.ServiceInfo;
import android.hardware.Sensor;
import android.hardware.SensorEvent;
import android.hardware.SensorEventListener;
import android.hardware.SensorManager;
import android.os.IBinder;
import android.os.PowerManager;
import android.util.Log;

import androidx.core.app.ServiceCompat;
import androidx.core.content.ContextCompat;

import com.university.app.util.Notifs;

public class AccelService extends Service implements SensorEventListener {
    private static final int PERIOD_SCREEN_ON_US = 20_000;   // 50 Гц
    private static final int PERIOD_SCREEN_OFF_US = 200_000; // 5 Гц, в 10 раз реже

    private SensorManager sensorManager;
    private Sensor accelerometer;
    private int samples = 0;

    private final BroadcastReceiver screenReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            boolean screenOff = Intent.ACTION_SCREEN_OFF.equals(intent.getAction());
            register(screenOff ? PERIOD_SCREEN_OFF_US : PERIOD_SCREEN_ON_US);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        sensorManager = getSystemService(SensorManager.class);
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
        IntentFilter filter = new IntentFilter(Intent.ACTION_SCREEN_OFF);
        filter.addAction(Intent.ACTION_SCREEN_ON);
        // SCREEN_ON/OFF принимаются только динамически
        ContextCompat.registerReceiver(this, screenReceiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ServiceCompat.startForeground(this, 348,
                Notifs.build(this, "Датчик движения", "Экономный режим при выключенном экране"),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE);
        boolean interactive = getSystemService(PowerManager.class).isInteractive();
        register(interactive ? PERIOD_SCREEN_ON_US : PERIOD_SCREEN_OFF_US); // учитываем текущее состояние
        return START_STICKY;
    }

    private void register(int periodUs) {
        if (accelerometer == null) {
            stopSelf();
            return;
        }
        sensorManager.unregisterListener(this);
        sensorManager.registerListener(this, accelerometer, periodUs);
        Log.d("AccelService", "Период опроса: " + periodUs / 1000 + " мс");
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        if (++samples % 100 == 0) {
            Log.d("AccelService", "Отсчётов: " + samples);
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) { }

    @Override
    public void onDestroy() {
        unregisterReceiver(screenReceiver);
        sensorManager.unregisterListener(this);
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
Проверка: смотрите в Logcat «Период опроса: 20 мс», а после блокировки экрана «Период опроса: 200 мс». Система может использовать указанный период как подсказку, но реальная частота зависит от датчика.

---

### 49. Временные права на URI для стороннего редактора

Вместо открытия доступа к файлу навсегда даём редактору временные права на **один** URI. Флаги работают вместе с `ClipData` (так права корректно передаются и через `createChooser`, и на старых версиях Android).

```xml
<queries>
    <intent>
        <action android:name="android.intent.action.EDIT" />
        <data android:mimeType="image/*" />
    </intent>
</queries>
```
(`FileProvider` и `file_paths.xml` — в начале этого файла.)

```java
package com.university.app;

import android.content.ActivityNotFoundException;
import android.content.ClipData;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.widget.Button;
import android.widget.Toast;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;

import java.io.File;

public class EditPhotoActivity extends AppCompatActivity {
    private Uri sharedUri;

    private final ActivityResultLauncher<Intent> editor =
            registerForActivityResult(new ActivityResultContracts.StartActivityForResult(), result -> {
                if (sharedUri != null) {
                    // Права больше не нужны — отзываем сразу после возврата из редактора
                    revokeUriPermission(sharedUri,
                            Intent.FLAG_GRANT_READ_URI_PERMISSION | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
                    sharedUri = null;
                }
            });

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Button button = new Button(this);
        button.setText("Открыть фото в редакторе");
        button.setOnClickListener(v -> openInEditor(new File(getFilesDir(), "photos/sample.jpg")));
        setContentView(button);
    }

    private void openInEditor(File photo) {
        if (!photo.exists()) {
            Toast.makeText(this, "Нет файла photos/sample.jpg", Toast.LENGTH_SHORT).show();
            return;
        }
        sharedUri = FileProvider.getUriForFile(this, getPackageName() + ".fileprovider", photo);

        Intent edit = new Intent(Intent.ACTION_EDIT);
        edit.setDataAndType(sharedUri, "image/*");
        edit.setClipData(ClipData.newRawUri("photo", sharedUri)); // чтобы права дошли и через chooser
        edit.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION          // чтение
                | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);           // и запись результата правок
        try {
            editor.launch(Intent.createChooser(edit, "Редактировать фото"));
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, "Нет приложения-редактора", Toast.LENGTH_SHORT).show();
        }
    }
}
```
Важные детали: права действуют только на этот URI, привязаны к получающему приложению и исчезают при завершении его задачи (и при нашем `revokeUriPermission`). Если надо выдать права конкретному пакету заранее: `grantUriPermission("com.package.editor", uri, flags)`.

---

### 50. Комплексная Enterprise-архитектура

Состав: защищённый Foreground Service (`dataSync`) → локальный `ItemsProvider` с постраничной выдачей → динамический `BroadcastReceiver` сетевого статуса внутри сервиса → полная очистка ресурсов в `onDestroy()`.

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service android:name=".enterprise.EnterpriseSyncService" android:exported="false"
    android:foregroundServiceType="dataSync" />
<provider android:name=".enterprise.ItemsProvider"
    android:authorities="com.university.app.enterprise.items" android:exported="false" />
```

**Постраничный провайдер** (лимит и смещение приходят в стандартных аргументах `QUERY_ARG_LIMIT`/`QUERY_ARG_OFFSET`):

```java
package com.university.app.enterprise;

import android.content.ContentProvider;
import android.content.ContentResolver;
import android.content.ContentValues;
import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;
import android.net.Uri;
import android.os.Bundle;
import android.os.CancellationSignal;

public class ItemsProvider extends ContentProvider {
    public static final String AUTHORITY = "com.university.app.enterprise.items";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/items");
    private static final int DEFAULT_LIMIT = 50;
    private static final int MAX_LIMIT = 200;

    private static class Db extends SQLiteOpenHelper {
        Db(Context c) {
            super(c, "enterprise.db", null, 1);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL("CREATE TABLE items (_id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL)");
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int o, int n) {
            db.execSQL("DROP TABLE IF EXISTS items");
            onCreate(db);
        }
    }

    private Db helper;

    @Override
    public boolean onCreate() {
        helper = new Db(getContext()); // ничего тяжёлого: БД открывается лениво
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, Bundle queryArgs, CancellationSignal signal) {
        int limit = Math.min(MAX_LIMIT, Math.max(1,
                queryArgs.getInt(ContentResolver.QUERY_ARG_LIMIT, DEFAULT_LIMIT)));
        int offset = Math.max(0, queryArgs.getInt(ContentResolver.QUERY_ARG_OFFSET, 0));

        SQLiteDatabase db = helper.getReadableDatabase();
        Cursor c = db.query("items", projection, null, null, null, null, "_id ASC",
                offset + "," + limit); // LIMIT offset,limit

        long total = android.database.DatabaseUtils.queryNumEntries(db, "items");
        Bundle extras = new Bundle();
        extras.putInt(ContentResolver.EXTRA_TOTAL_COUNT, (int) total); // клиенту — сколько всего страниц
        c.setExtras(extras);
        c.setNotificationUri(getContext().getContentResolver(), CONTENT_URI);
        return c;
    }

    // Старый вариант сигнатуры тоже должен работать
    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        return query(uri, projection, new Bundle(), null);
    }

    @Override
    public int bulkInsert(Uri uri, ContentValues[] values) {
        SQLiteDatabase db = helper.getWritableDatabase();
        db.beginTransaction();
        try {
            for (ContentValues v : values) {
                db.insertOrThrow("items", null, v);
            }
            db.setTransactionSuccessful();
        } finally {
            db.endTransaction();
        }
        getContext().getContentResolver().notifyChange(CONTENT_URI, null);
        return values.length;
    }

    @Override public String getType(Uri uri) { return "vnd.android.cursor.dir/vnd.com.university.item"; }
    @Override public Uri insert(Uri uri, ContentValues values) { throw new UnsupportedOperationException("Используйте bulkInsert"); }
    @Override public int delete(Uri uri, String s, String[] a) { throw new UnsupportedOperationException(); }
    @Override public int update(Uri uri, ContentValues v, String s, String[] a) { throw new UnsupportedOperationException(); }
}
```

**Динамический приёмник сетевого статуса:**

```java
package com.university.app.enterprise;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.net.ConnectivityManager;
import android.net.Network;
import android.net.NetworkCapabilities;
import android.util.Log;

public class NetworkStatusReceiver extends BroadcastReceiver {
    private volatile boolean online = true;

    public boolean isOnline() {
        return online;
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        ConnectivityManager cm = context.getSystemService(ConnectivityManager.class);
        Network net = cm.getActiveNetwork();
        NetworkCapabilities caps = (net == null) ? null : cm.getNetworkCapabilities(net);
        online = caps != null && caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED);
        Log.d("NetStatus", online ? "Сеть доступна" : "Сети нет");
    }
}
```

**Сервис синхронизации:**

```java
package com.university.app.enterprise;

import android.app.NotificationManager;
import android.app.Service;
import android.content.ContentValues;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.ServiceInfo;
import android.net.ConnectivityManager;
import android.util.Log;
import android.os.IBinder;

import androidx.core.app.ServiceCompat;
import androidx.core.content.ContextCompat;

import com.university.app.util.Notifs;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class EnterpriseSyncService extends Service {
    private static final String TAG = "EnterpriseSync";
    private static final int NOTIF_ID = 350;
    private static final int PAGES = 20;
    private static final int PAGE_SIZE = 100;

    private ExecutorService executor;
    private NetworkStatusReceiver networkReceiver;
    private volatile boolean stopped = false;

    @Override
    public void onCreate() {
        super.onCreate();
        executor = Executors.newSingleThreadExecutor();
        networkReceiver = new NetworkStatusReceiver();
        // CONNECTIVITY_ACTION устарел, но динамическим приёмникам доставляется
        ContextCompat.registerReceiver(this, networkReceiver,
                new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION),
                ContextCompat.RECEIVER_NOT_EXPORTED);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Первым делом — foreground, чтобы уложиться в 5 секунд и не получить краш/ANR
        ServiceCompat.startForeground(this, NOTIF_ID,
                Notifs.build(this, "Корпоративная синхронизация", "Старт..."),
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC);
        stopped = false;
        executor.execute(() -> sync(startId));
        return START_REDELIVER_INTENT;
    }

    private void sync(int startId) {
        for (int page = 0; page < PAGES && !stopped; page++) {
            while (!networkReceiver.isOnline() && !stopped) { // нет сети — ждём, но не крутим CPU
                sleep(2000);
            }
            if (stopped) break;
            ContentValues[] rows = new ContentValues[PAGE_SIZE];
            for (int i = 0; i < PAGE_SIZE; i++) { // здесь сетевой запрос страницы
                rows[i] = new ContentValues();
                rows[i].put("name", "Запись " + (page * PAGE_SIZE + i + 1));
            }
            getContentResolver().bulkInsert(ItemsProvider.CONTENT_URI, rows);
            getSystemService(NotificationManager.class).notify(NOTIF_ID,
                    Notifs.build(this, "Корпоративная синхронизация",
                            "Страница " + (page + 1) + " из " + PAGES));
            sleep(500);
        }
        Log.d(TAG, "Синхронизация завершена");
        ServiceCompat.stopForeground(this, ServiceCompat.STOP_FOREGROUND_REMOVE);
        stopSelf(startId);
    }

    /** Android 15+: система вызывает при исчерпании лимита времени для dataSync. Нужно остановиться. */
    @Override
    public void onTimeout(int startId, int fgsType) {
        Log.w(TAG, "Лимит времени foreground-сервиса исчерпан — останавливаемся");
        stopped = true;
        stopSelf(startId);
    }

    private void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            stopped = true;
        }
    }

    @Override
    public void onDestroy() {
        stopped = true;
        unregisterReceiver(networkReceiver); // 1) приёмник
        executor.shutdownNow();              // 2) поток (прерываем sleep) без блокировки главного потока
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

Чтение страницы клиентом:
```java
Bundle args = new Bundle();
args.putInt(ContentResolver.QUERY_ARG_LIMIT, 50);
args.putInt(ContentResolver.QUERY_ARG_OFFSET, 100); // третья страница
try (Cursor c = getContentResolver().query(ItemsProvider.CONTENT_URI,
        new String[]{"_id", "name"}, args, null)) {
    int total = c.getExtras().getInt(ContentResolver.EXTRA_TOTAL_COUNT);
    android.util.Log.d("Page", "Всего записей: " + total + ", на странице: " + c.getCount());
}
```

**Про требование «нулевая вероятность ANR».** Гарантировать **нулевую** вероятность нельзя ни в одном приложении: ANR вызывают и факторы вне вашего кода (перегруженное устройство, блокировки системных сервисов, другие процессы). Реально можно добиться того, что ваш код ANR не провоцирует. В решении это обеспечено так:

- `ItemsProvider.onCreate()` и `Service.onCreate()` не делают ничего тяжёлого;
- `startForeground()` вызывается первым действием в `onStartCommand()`;
- вся работа (сеть, БД, ожидание) идёт в отдельном потоке `executor`, главный поток свободен;
- в `NetworkStatusReceiver.onReceive()` только лёгкие вызовы, никакого ввода-вывода;
- ожидание сети реализовано через `sleep` в фоновом потоке, а не через блокировку главного;
- реализован `onTimeout()` для Android 15, иначе по истечении лимита система остановит процесс с ошибкой;
- в `onDestroy()` снимается приёмник и прерывается поток (`shutdownNow()`), главный поток не ждёт завершения.

Проверка на практике: включите в debug-сборке `StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().detectAll().penaltyLog().build())` и убедитесь, что в Logcat нет предупреждений о диске/сети на главном потоке. Для Android 15 учтите ещё два ограничения `dataSync`: суммарный лимит около 6 часов в сутки и запрет запуска из `BOOT_COMPLETED`.
