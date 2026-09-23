# Робота з мережею в React Native

## Зміст

1. [Як мобільний застосунок отримує дані](#як-мобільний-застосунок-отримує-дані)
2. [Fetch API](#fetch-api)
3. [Обробка помилок](#обробка-помилок)
4. [Завантаження даних у компоненті](#завантаження-даних-у-компоненті)
5. [Скасування запитів](#скасування-запитів)
6. [Відправлення даних на сервер](#відправлення-даних-на-сервер)
7. [Pull-to-refresh](#pull-to-refresh)
8. [Винесення запитів в окремий шар](#винесення-запитів-в-окремий-шар)
9. [Адреса API та змінні оточення](#адреса-api-та-змінні-оточення)
10. [Корисні ресурси](#корисні-ресурси)

---

## Як мобільний застосунок отримує дані

Більшість реальних застосунків не зберігає всі дані локально: список товарів, стрічка новин, профіль користувача приходять із сервера через **HTTP-запити** до **API** (Application Programming Interface).

Типовий обмін виглядає так:

```
Застосунок  ──  GET /posts  ──────────────▶  Сервер
Застосунок  ◀──  200 OK + JSON-масив  ─────  Сервер
```

Основні HTTP-методи:

| Метод | Призначення | Приклад |
|---|---|---|
| `GET` | отримати дані | список постів, профіль |
| `POST` | створити новий запис | новий пост, реєстрація |
| `PUT` / `PATCH` | оновити запис повністю / частково | редагування профілю |
| `DELETE` | видалити запис | видалення коментаря |

Сервер відповідає **статус-кодом**, який варто завжди перевіряти:

| Код | Значення |
|---|---|
| `200`–`299` | успіх (`200 OK`, `201 Created`, `204 No Content`) |
| `400` | некоректний запит (помилка валідації) |
| `401` / `403` | не авторизований / немає доступу |
| `404` | ресурс не знайдено |
| `500`–`599` | помилка на боці сервера |

Для навчальних прикладів у цьому конспекті використовується безкоштовний тестовий API **JSONPlaceholder** (`https://jsonplaceholder.typicode.com`). Він повертає реалістичні дані (пости, користувачі, коментарі) і приймає `POST`/`PUT`/`DELETE`, але **не зберігає зміни насправді** — сервер лише імітує успішну відповідь.

Тип даних, з яким працюватимуть приклади:

```ts
// types/post.ts
export type Post = {
  id: number;
  userId: number;
  title: string;
  body: string;
};
```

---

## Fetch API

`fetch` — вбудована функція для HTTP-запитів. У React Native вона працює так само, як у браузері, встановлювати нічого не потрібно.

### Найпростіший запит

```ts
fetch('https://jsonplaceholder.typicode.com/posts/1')
  .then((response) => response.json())
  .then((data) => console.log(data));
```

`fetch` повертає **Promise** з об'єктом `Response`. Тіло відповіді ще треба окремо "розпакувати" — `response.json()` теж повертає Promise, тому в ланцюжку два `.then()`.

### Те саме через async/await

```ts
async function loadPost() {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts/1');
  const data = await response.json();
  console.log(data);
}
```

Синтаксис `async/await` читається як звичайний послідовний код, тому далі в конспекті використовується саме він.

### Що є в об'єкті Response

| Властивість / метод | Опис |
|---|---|
| `response.ok` | `true`, якщо статус у діапазоні 200–299 |
| `response.status` | числовий код (`200`, `404`, `500`) |
| `response.headers` | заголовки відповіді |
| `response.json()` | розібрати тіло як JSON |
| `response.text()` | отримати тіло як рядок |

### Параметри запиту

Другий аргумент `fetch` — об'єкт налаштувань:

```ts
const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    Authorization: 'Bearer <токен>',
  },
  body: JSON.stringify({ title: 'Новий пост', body: 'Текст', userId: 1 }),
});
```

- `method` — HTTP-метод (за замовчуванням `GET`).
- `headers` — заголовки: тип вмісту, токен авторизації тощо.
- `body` — тіло запиту. Об'єкт обов'язково перетворюється на рядок через `JSON.stringify`.

### Query-параметри в URL

Для фільтрів і пагінації параметри додаються до адреси. Щоб не склеювати рядок вручну й коректно екранувати спецсимволи, зручно використати `URLSearchParams`:

```ts
const params = new URLSearchParams({ _page: '2', _limit: '10' });
const response = await fetch(`https://jsonplaceholder.typicode.com/posts?${params}`);
// → /posts?_page=2&_limit=10
```

---

## Обробка помилок

### fetch не вважає помилкою статус 404 чи 500

Це найважливіша особливість `fetch`: Promise **відхиляється лише при мережевій помилці** (немає інтернету, сервер недоступний). Якщо сервер відповів `404` чи `500`, `fetch` вважає це успішним запитом — відповідь же прийшла.

Тому статус треба перевіряти вручну через `response.ok`:

```ts
async function getPost(id: number): Promise<Post> {
  const response = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);

  if (!response.ok) {
    throw new Error(`Помилка сервера: ${response.status}`);
  }

  return response.json();
}
```

### try/catch/finally

```ts
try {
  const post = await getPost(1);
  console.log(post.title);
} catch (error) {
  // сюди потрапляють і мережеві помилки, і наш throw при !response.ok
  console.log('Не вдалося завантажити:', error);
} finally {
  // виконується завжди — зручно для вимкнення індикатора завантаження
}
```

### Власний клас помилки зі статусом

Щоб у інтерфейсі розрізняти "не знайдено", "немає доступу" і "сервер впав", корисно зберігати статус у помилці:

```ts
export class ApiError extends Error {
  status: number;

  constructor(status: number, message: string) {
    super(message);
    this.status = status;
  }
}

// використання
if (error instanceof ApiError && error.status === 404) {
  setMessage('Пост не знайдено');
}
```

### Таймаут запиту

`fetch` не має вбудованого таймауту: на поганому мобільному з'єднанні запит може "висіти" дуже довго. Таймаут реалізується через `AbortController` (детальніше — у розділі про скасування):

```ts
async function fetchWithTimeout(url: string, ms = 10000) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), ms);

  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(timer);
  }
}
```

---

## Завантаження даних у компоненті

Класичний підхід без бібліотек: `useEffect` запускає запит при появі екрана, а три змінні стану описують, що зараз показувати.

### Три стани екрана

| Стан | Що показувати |
|---|---|
| `loading` | індикатор завантаження |
| `error` | повідомлення про помилку + кнопка "Спробувати ще" |
| `success` | самі дані (або "Список порожній") |

### Повний приклад

```tsx
import { useEffect, useState } from 'react';
import { View, Text, FlatList, ActivityIndicator, Pressable, StyleSheet } from 'react-native';
import type { Post } from '../types/post';

export default function PostsScreen() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const loadPosts = async () => {
    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=20');
      if (!response.ok) throw new Error(`Статус ${response.status}`);
      const data: Post[] = await response.json();
      setPosts(data);
    } catch (e) {
      setError('Не вдалося завантажити пости');
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    loadPosts();
  }, []);

  if (isLoading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  if (error) {
    return (
      <View style={styles.center}>
        <Text style={styles.error}>{error}</Text>
        <Pressable style={styles.button} onPress={loadPosts}>
          <Text style={styles.buttonText}>Спробувати ще</Text>
        </Pressable>
      </View>
    );
  }

  return (
    <FlatList
      data={posts}
      keyExtractor={(item) => String(item.id)}
      contentContainerStyle={styles.list}
      ListEmptyComponent={<Text style={styles.empty}>Постів поки немає</Text>}
      renderItem={({ item }) => (
        <View style={styles.card}>
          <Text style={styles.title}>{item.title}</Text>
          <Text numberOfLines={2} style={styles.body}>{item.body}</Text>
        </View>
      )}
    />
  );
}

const styles = StyleSheet.create({
  center: { flex: 1, justifyContent: 'center', alignItems: 'center', gap: 12, padding: 20 },
  error: { color: '#ff3b30', fontSize: 15, textAlign: 'center' },
  button: { backgroundColor: '#007AFF', paddingVertical: 10, paddingHorizontal: 20, borderRadius: 8 },
  buttonText: { color: '#fff', fontWeight: '600' },
  list: { padding: 16, gap: 12 },
  empty: { textAlign: 'center', color: '#8e8e93', marginTop: 40 },
  card: { backgroundColor: '#fff', padding: 16, borderRadius: 12, gap: 6 },
  title: { fontSize: 16, fontWeight: '700' },
  body: { fontSize: 14, color: '#6e6e73' },
});
```

### Чому функція в useEffect не async напряму

```tsx
// ❌ так не можна
useEffect(async () => { ... }, []);

// ✅ так
useEffect(() => {
  loadPosts();
}, []);
```

`useEffect` очікує, що колбек поверне або нічого, або функцію очищення. `async`-функція завжди повертає Promise, тому її викликають усередині звичайного колбека.

### ActivityIndicator

`ActivityIndicator` — вбудований системний спінер, який на iOS і Android виглядає "рідно" для кожної платформи:

```tsx
<ActivityIndicator size="large" color="#007AFF" />
```

---

## Скасування запитів

Проблема: користувач відкрив екран, запит пішов, але користувач одразу повернувся назад. Відповідь приходить, коли компонента вже немає — у кращому випадку це марна робота, у гіршому — спроба оновити стан розмонтованого компонента.

Ще гірший випадок — **стан гонитви (race condition)**: у полі пошуку користувач швидко вводить "ca", потім "cat". Відповідь на "ca" може прийти **пізніше** за відповідь на "cat" і перезаписати правильні результати неправильними.

Обидві проблеми вирішує `AbortController`:

```tsx
useEffect(() => {
  const controller = new AbortController();

  const load = async () => {
    try {
      const response = await fetch(
        `https://jsonplaceholder.typicode.com/posts?q=${encodeURIComponent(query)}`,
        { signal: controller.signal }
      );
      const data = await response.json();
      setResults(data);
    } catch (e) {
      if (e instanceof Error && e.name === 'AbortError') return; // запит скасовано — це не помилка
      setError('Помилка пошуку');
    }
  };

  load();

  // функція очищення: спрацює при зміні query або при розмонтуванні
  return () => controller.abort();
}, [query]);
```

Коли `query` змінюється, React спочатку викликає функцію очищення попереднього ефекту (`controller.abort()` скасовує старий запит), а потім запускає новий. Застаріла відповідь ніколи не потрапить у стан.

### Debounce для пошуку

Щоб не відправляти запит на кожну натиснуту літеру, запит відкладають, доки користувач не зробить паузу у введенні:

```tsx
const [query, setQuery] = useState('');
const [debouncedQuery, setDebouncedQuery] = useState('');

useEffect(() => {
  const timer = setTimeout(() => setDebouncedQuery(query), 400);
  return () => clearTimeout(timer);
}, [query]);

// запит виконується в useEffect, що залежить від debouncedQuery, а не від query
```

---

## Відправлення даних на сервер

### POST — створення запису

```tsx
import { useState } from 'react';
import { View, TextInput, Pressable, Text, Alert, ActivityIndicator, StyleSheet } from 'react-native';

export default function CreatePostScreen() {
  const [title, setTitle] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async () => {
    if (!title.trim()) {
      Alert.alert('Помилка', 'Введіть заголовок');
      return;
    }

    setIsSubmitting(true);
    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title: title.trim(), body: '...', userId: 1 }),
      });

      if (!response.ok) throw new Error(`Статус ${response.status}`);

      const created = await response.json();
      Alert.alert('Готово', `Пост створено з id ${created.id}`);
      setTitle('');
    } catch (e) {
      Alert.alert('Помилка', 'Не вдалося створити пост');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <View style={styles.container}>
      <TextInput
        style={styles.input}
        placeholder="Заголовок поста"
        value={title}
        onChangeText={setTitle}
        editable={!isSubmitting}
      />
      <Pressable
        style={[styles.button, isSubmitting && styles.buttonDisabled]}
        onPress={handleSubmit}
        disabled={isSubmitting}
      >
        {isSubmitting ? (
          <ActivityIndicator color="#fff" />
        ) : (
          <Text style={styles.buttonText}>Опублікувати</Text>
        )}
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 20, gap: 12 },
  input: { borderWidth: 1, borderColor: '#c7c7cc', borderRadius: 8, padding: 12, fontSize: 16 },
  button: { backgroundColor: '#007AFF', padding: 14, borderRadius: 8, alignItems: 'center' },
  buttonDisabled: { opacity: 0.6 },
  buttonText: { color: '#fff', fontWeight: '600' },
});
```

Кнопка блокується на час запиту (`disabled={isSubmitting}`). Без цього нетерплячий користувач натисне кілька разів і створить кілька однакових записів.

### PUT, PATCH, DELETE

```ts
// повне оновлення
await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ id, title, body, userId }),
});

// часткове оновлення
await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title }),
});

// видалення
await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, { method: 'DELETE' });
```

### Відправлення файлу (FormData)

Для завантаження зображення (наприклад, аватара) використовується `FormData`. У React Native файл описується об'єктом з `uri`, `name` і `type`:

```ts
const formData = new FormData();
formData.append('avatar', {
  uri: imageUri,          // наприклад, з expo-image-picker
  name: 'avatar.jpg',
  type: 'image/jpeg',
} as any);

await fetch('https://example.com/api/upload', {
  method: 'POST',
  body: formData,
  // Content-Type не вказується вручну: fetch сам додасть multipart/form-data з boundary
});
```

---

## Pull-to-refresh

"Потягни вниз, щоб оновити" — стандартний мобільний патерн. `FlatList` підтримує його через два пропси:

```tsx
const [isRefreshing, setIsRefreshing] = useState(false);

const handleRefresh = async () => {
  setIsRefreshing(true);
  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=20');
    const data: Post[] = await response.json();
    setPosts(data);
  } finally {
    setIsRefreshing(false);
  }
};

<FlatList
  data={posts}
  keyExtractor={(item) => String(item.id)}
  renderItem={renderPost}
  refreshing={isRefreshing}
  onRefresh={handleRefresh}
/>
```

Окремий стан `isRefreshing` потрібен, щоб при оновленні не показувати повноекранний спінер (як при першому завантаженні): користувач бачить старий список і маленький системний індикатор зверху.

Для `ScrollView` те саме робиться через компонент `RefreshControl`:

```tsx
import { ScrollView, RefreshControl } from 'react-native';

<ScrollView
  refreshControl={<RefreshControl refreshing={isRefreshing} onRefresh={handleRefresh} />}
>
  {/* контент */}
</ScrollView>
```

---

## Винесення запитів в окремий шар

Коли `fetch` з URL-адресами розкиданий по всіх екранах, будь-яка зміна (нова адреса сервера, заголовок авторизації, формат помилок) вимагає правок у десятках місць. Правильніше зібрати мережеву логіку в окремій папці.

```
api/
├── client.ts     ← базова функція запиту
└── posts.ts      ← функції для конкретного ресурсу
```

### Базовий клієнт

```ts
// api/client.ts
const BASE_URL = process.env.EXPO_PUBLIC_API_URL ?? 'https://jsonplaceholder.typicode.com';

export class ApiError extends Error {
  status: number;

  constructor(status: number, message: string) {
    super(message);
    this.status = status;
  }
}

export async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
  const response = await fetch(`${BASE_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  if (!response.ok) {
    throw new ApiError(response.status, `Запит ${path} завершився зі статусом ${response.status}`);
  }

  // 204 No Content — тіла немає, json() викинув би помилку
  if (response.status === 204) {
    return undefined as T;
  }

  return response.json() as Promise<T>;
}
```

### Функції ресурсу

```ts
// api/posts.ts
import { request } from './client';
import type { Post } from '../types/post';

export type NewPost = Omit<Post, 'id'>;

export const getPosts = (page = 1, limit = 10, signal?: AbortSignal) =>
  request<Post[]>(`/posts?_page=${page}&_limit=${limit}`, { signal });

export const getPost = (id: number | string, signal?: AbortSignal) =>
  request<Post>(`/posts/${id}`, { signal });

export const createPost = (data: NewPost) =>
  request<Post>('/posts', { method: 'POST', body: JSON.stringify(data) });

export const updatePost = (id: number, data: Partial<NewPost>) =>
  request<Post>(`/posts/${id}`, { method: 'PATCH', body: JSON.stringify(data) });

export const deletePost = (id: number) =>
  request<void>(`/posts/${id}`, { method: 'DELETE' });
```

Тепер екрани не знають нічого про URL, заголовки чи статус-коди:

```ts
const posts = await getPosts(1, 20);
```

Цей шар стане основою і для TanStack Query: бібліотека не виконує запити сама, а викликає саме такі функції.

---

## Адреса API та змінні оточення

Адреса сервера зазвичай відрізняється для розробки й продакшену. Жорстко прописувати її в коді незручно — для цього існують **змінні оточення**.

Expo автоматично підхоплює змінні з файлу `.env` у корені проєкту, якщо їхня назва починається з префікса `EXPO_PUBLIC_`:

```bash
# .env
EXPO_PUBLIC_API_URL=https://jsonplaceholder.typicode.com
```

```ts
const BASE_URL = process.env.EXPO_PUBLIC_API_URL;
```

Важливі моменти:

- Змінна має звертатися **саме** як `process.env.EXPO_PUBLIC_API_URL` (крапковий запис). Деструктуризація `const { EXPO_PUBLIC_API_URL } = process.env` не спрацює, бо значення підставляються в код на етапі збірки.
- Після зміни `.env` варто перезапустити Metro.
- Змінні з префіксом `EXPO_PUBLIC_` **вбудовуються в код застосунку** й доступні будь-кому, хто розбере зібраний файл. Секретні ключі (наприклад, ключ платіжної системи) у мобільному застосунку зберігати не можна — вони мають жити на сервері.
- Файли `.env*.local` зазвичай додаються в `.gitignore`.

---

## Корисні ресурси

- [Networking — React Native Docs](https://reactnative.dev/docs/network) — офіційний огляд роботи з мережею в React Native.
- [Fetch API — MDN](https://developer.mozilla.org/uk/docs/Web/API/Fetch_API) — повний довідник `fetch`, `Request`, `Response`.
- [AbortController — MDN](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) — скасування запитів.
- [URLSearchParams — MDN](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) — робота з query-параметрами.
- [FlatList — React Native Docs](https://reactnative.dev/docs/flatlist) — пропси `onRefresh`, `onEndReached`, `ListFooterComponent`.
- [RefreshControl — React Native Docs](https://reactnative.dev/docs/refreshcontrol) — pull-to-refresh для `ScrollView`.
- [Environment variables — Expo Docs](https://docs.expo.dev/guides/environment-variables/) — змінні `EXPO_PUBLIC_` і файли `.env`.
- [expo-network — Expo Docs](https://docs.expo.dev/versions/latest/sdk/network/) — стан мережевого з'єднання.
- [Axios — документація](https://axios-http.com/docs/intro) — альтернатива `fetch`.
- [JSONPlaceholder](https://jsonplaceholder.typicode.com) — безкоштовний тестовий API для прикладів.
