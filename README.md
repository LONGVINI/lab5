<h2>Звіт по лабораторній роботі: Розробка додатку для візуалізації вимірювань GPS</h2>

<h3>Мета роботи</h3>
<p>Метою даної лабораторної роботи є розробка додатку, який зчитує дані з емульованої вимірювальної частини GPS, наданої у вигляді Docker image, та відображає положення об'єкта і супутників на графіку в декартових координатах.</p>

<h3>Теоретичні відомості</h3>
<p><strong>Трилатерація</strong> — це метод визначення положення об'єкта у просторі на основі відстані до кількох точок з відомими координатами. Цей метод активно використовується в GPS для обчислення місцезнаходження.</p>
<p>Для розрахунку положення об'єкта використовується система рівнянь, яка описує кола (у 2D) або сфери (у 3D) навколо відомих точок. Положення об'єкта визначається в точці перетину цих кіл або сфер.</p>

<h3>Методика розв'язання</h3>
<p>В лабораторній роботі використовуються три точки з відомими координатами та відстані до шуканого об'єкта:</p>
<ul>
  <li>Точки: A(x<sub>1</sub>, y<sub>1</sub>), B(x<sub>2</sub>, y<sub>2</sub>), C(x<sub>3</sub>, y<sub>3</sub>).</li>
  <li>Відстані: r<sub>1</sub>, r<sub>2</sub>, r<sub>3</sub>.</li>
</ul>
<p>Рівняння кіл для цих точок мають вигляд:</p>
<ul>
  <li>(x - x<sub>1</sub>)² + (y - y<sub>1</sub>)² = r<sub>1</sub>²</li>
  <li>(x - x<sub>2</sub>)² + (y - y<sub>2</sub>)² = r<sub>2</sub>²</li>
  <li>(x - x<sub>3</sub>)² + (y - y<sub>3</sub>)² = r<sub>3</sub>²</li>
</ul>

<h3>Розробка додатку</h3>
<p>1. Завантаження Docker-образу GPS-емулятора за допомогою команди:</p>
<pre><code>docker pull your-gps-emulator-image</code></pre>
<p>2. Запуск контейнера для емуляції даних GPS:</p>
<pre><code>docker run --name gps-emulator -p 4000:4000 your-gps-emulator-image</code></pre>
<p>3. Отримання даних через WebSocket та використання бібліотеки для візуалізації координат об'єкта і супутників.</p>

<h3>Результат</h3>
<h3>Розробка додатку</h3>

<p>Для реалізації взаємодії з GPS-емулятором був розроблений WebSocket-клієнт на Unity. Цей код підключається до WebSocket-сервера, отримує дані про координати об'єкта в реальному часі та відправляє їх на обробку для подальшої візуалізації. Клієнт обробляє повідомлення, що приходять у форматі JSON, і передає їх для відображення супутникових координат у додатку.</p>

<p>Клієнт включає наступні основні функції:</p>
<ul>
    <li>Підключення до WebSocket сервера через порт 4001;</li>
    <li>Обробка підключення, помилок і закриття з'єднання;</li>
    <li>Отримання повідомлень з даними GPS у форматі JSON;</li>
    <li>Передача даних для подальшої обробки й візуалізації у додатку;</li>
</ul>

<p>Нижче наведений код для підключення та обробки даних:</p>

```csharp
using System;
using System.Text;
using System.Threading.Tasks;
using UnityEngine;
using NativeWebSocket;

public class WebSocketClient : MonoBehaviour
{
    private WebSocket websocket;

    async void Start()
    {
        websocket = new WebSocket("ws://localhost:4001");

        websocket.OnOpen += () => {  Debug.Log("Соединение установлено!"); };
        websocket.OnError += (e) => { Debug.LogError("Ошибка: " + e); };
        websocket.OnClose += (e) => { Debug.Log("Соединение закрыто!");};
        websocket.OnMessage += (bytes) => {
            var message = Encoding.UTF8.GetString(bytes);
            ProcessGpsMessage(message);
        };

        await websocket.Connect();
    }

    void Update()
    {
#if !UNITY_WEBGL || UNITY_EDITOR
        websocket?.DispatchMessageQueue();
#endif
    }

    async void OnApplicationQuit()
    {
        await websocket.Close();
    }

    private void ProcessGpsMessage(string message)
    {
        GpsMessage gpsMessage = JsonUtility.FromJson<GpsMessage>(message);
        Debug.Log($"ID: {gpsMessage.id}, X: {gpsMessage.x}, Y: {gpsMessage.y}, SentAt: {gpsMessage.sentAt}, ReceivedAt: {gpsMessage.receivedAt}");

        SatelliteManager.Instance.UpdateSatellitePosition(gpsMessage.id, gpsMessage.x, gpsMessage.y, gpsMessage.sentAt, gpsMessage.receivedAt);
    }
}


[Serializable]
public class GpsMessage
{
    public string id;
    public float x;
    public float y;
    public double sentAt;
    public double receivedAt;
}
```

<p>Клас <code>GraphDrawer</code> відповідає за візуалізацію графіка в Unity. Він створює сітку з ліній у декартових координатах, відображаючи координати об'єктів, отриманих з GPS-даних. Основні функції класу:</p>
<ul>
    <li>Інстанціює префаби для ліній та тексту;</li>
    <li>Використовує метод <code>DrawGraph</code> для побудови графіка;</li>
    <li>Встановлює позиції ліній і відповідних текстів, відображаючи координати.</li>
</ul>
<p>Нижче наведено код класу:</p>

```csharp
using UnityEngine;
using TMPro;

public class GraphDrawer : MonoBehaviour
{
    public GameObject linePrefab; 
    public GameObject textPrefab; 
    public int numberOfLines = 10;
    public float spacing = 2f;

    void Start()
    {
        DrawGraph();
    }

    void DrawGraph()
    {
        for (int i = 0; i < numberOfLines; i++)
        {
            if (i == 0) { continue; }
            Vector3 linePosition = new Vector3(i * spacing, 0, 0);
            GameObject line = Instantiate(linePrefab, linePosition, Quaternion.Euler(90f, 0f, 0f));
            line.transform.localScale = new Vector3(1f, 1f, 5f); 
            GameObject text = Instantiate(textPrefab, linePosition + new Vector3(0, 0.5f, 0.5f), Quaternion.Euler(90f, 0f, 0f)); 
            TextMeshPro textMeshPro = text.GetComponent<TextMeshPro>();
            textMeshPro.text = linePosition.x.ToString("F0"); 
            textMeshPro.alignment = TextAlignmentOptions.Center; 
        }

        for (int i = 0; i < numberOfLines; i++)
        {
            if (i == 0) { continue; }
            Vector3 linePosition = new Vector3(0, 0, i * spacing);
            GameObject line = Instantiate(linePrefab, linePosition, Quaternion.Euler(90f, 0f, 0f)); 
            line.transform.localScale = new Vector3(1f, 1f, 5f); 
            GameObject text = Instantiate(textPrefab, linePosition + new Vector3(-0.5f, 0f, 0f), Quaternion.Euler(90f, 0f, 0f));
            TextMeshPro textMeshPro = text.GetComponent<TextMeshPro>();
            textMeshPro.text = linePosition.z.ToString("F0"); 
            textMeshPro.alignment = TextAlignmentOptions.Center;
        }
    }
}
```

<p>Клас <code>MoveAndScale</code> відповідає за управління камерою в Unity. Він дозволяє користувачу переміщати камеру та змінювати її масштаб за допомогою клавіш та прокрутки миші. Основні функції класу:</p>
<ul>
    <li>Переміщення камери в горизонтальному та вертикальному напрямках;</li>
    <li>Зміна розміру (масштабу) камери за допомогою прокрутки миші;</li>
    <li>Обмеження масштабу в межах визначених значень.</li>
</ul>
<p>Нижче наведено код класу:</p>

```csharp
using UnityEngine;

public class MoveAndScale : MonoBehaviour
{
    public float moveSpeed = 5f;
    private float minSize = 1f;
    private float maxSize = 50f;

    private Camera camera;

    void Start()
    {
        camera = GetComponent<Camera>(); // Получаем компонент камеры
    }

    void Update()
    {
        float moveX = Input.GetAxis("Horizontal") * moveSpeed * Time.deltaTime; 
        float moveZ = Input.GetAxis("Vertical") * moveSpeed * Time.deltaTime;
        transform.position += new Vector3(moveX, 0, moveZ);

        float scroll = Input.GetAxis("Mouse ScrollWheel");
        if (scroll != 0)
        {
            camera.orthographicSize = Mathf.Clamp(camera.orthographicSize - scroll * 5, minSize, maxSize);
        }
    }
}
```
<p>Клас <code>MoveAndScale</code> відповідає за управління камерою в Unity. Він дозволяє користувачу переміщати камеру в просторі за допомогою клавіатури та змінювати масштаб за допомогою прокрутки миші. Основні функції цього класу:</p>
<ul>
    <li>Переміщення камери в горизонтальному та вертикальному напрямках через клавіші;</li>
    <li>Зміна масштабу камери (зменшення або збільшення) за допомогою коліщатка миші;</li>
    <li>Обмеження зміни масштабу в межах мінімальних і максимальних значень.</li>
</ul>
<p>Нижче наведено код класу:</p>

```csharp
using UnityEngine;

public class MoveAndScale : MonoBehaviour
{
    public float moveSpeed = 5f;
    private float minSize = 1f;
    private float maxSize = 50f;

    private Camera camera;

    void Start()
    {
        camera = GetComponent<Camera>(); 
    }

    void Update()
    {
        // Переміщення камери
        float moveX = Input.GetAxis("Horizontal") * moveSpeed * Time.deltaTime; 
        float moveZ = Input.GetAxis("Vertical") * moveSpeed * Time.deltaTime;
        transform.position += new Vector3(moveX, 0, moveZ);

        // Зміна масштабу камери
        float scroll = Input.GetAxis("Mouse ScrollWheel");
        if (scroll != 0)
        {
            camera.orthographicSize = Mathf.Clamp(camera.orthographicSize - scroll * 5, minSize, maxSize);
        }
    }
}
```

