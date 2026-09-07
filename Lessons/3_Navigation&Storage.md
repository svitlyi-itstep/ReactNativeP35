# 3. Навігація, передача даних та локальне збереження

## Зміст

1. [Навігація в Expo Router: основи](#навігація-в-expo-router-основи)
2. [Stack-навігація: екран поверх екрана](#stack-навігація-екран-поверх-екрана)
3. [Динамічні маршрути](#3-динамічні-маршрути)
4. [Передача даних через route-параметри](#передача-даних-через-route-параметри)
5. [Query-параметри для кількох значень](#query-параметри-для-кількох-значень)
6. [Чому не варто передавати весь об'єкт напряму](#чому-не-варто-передавати-весь-обєкт-напряму)
7. [Глобальний стан через Context API](#глобальний-стан-через-context-api)
8. [Локальне збереження: AsyncStorage](#локальне-збереження-asyncstorage)
9. [Як обрати правильний підхід](#як-обрати-правильний-підхід)
10. [Комплексний приклад: список → деталі → збереження](#комплексний-приклад-список--деталі--збереження)

---

## Навігація в Expo Router: основи

**Expo Router** будує навігацію на основі файлової структури проєкту — за аналогією з Next.js. Кожен файл у папці `app/` автоматично стає окремим маршрутом/екраном.

Раніше вже розглядали найпростіший випадок — вкладки (`app/(tabs)/`). Тепер додається другий тип переходів: **Stack-навігація** — коли один екран відкривається "поверх" іншого (наприклад, список → деталі), а не через таб-бар знизу.

### Два способи перейти на інший екран

```tsx
import { router } from 'expo-router';
import { Link } from 'expo-router';

// 1. Програмний перехід (усередині функції-обробника)
router.push('/details');

// 2. Декларативний перехід (як звичайне посилання)
<Link href="/details">
  <Text>Перейти до деталей</Text>
</Link>
```

`router.push()` зручний, коли перехід має відбутися після певної логіки (наприклад, після валідації форми). `Link` зручний, коли це просто клікабельний елемент інтерфейсу.

### Повернення назад

```tsx
router.back();
```

---

## Stack-навігація: екран поверх екрана

Якщо екран має бути доступним не через таб-бар, а через перехід з іншого екрана (наприклад, "деталі товару"), файл кладеться **поза** папкою `(tabs)`, просто в `app/`:

```
app/
├── (tabs)/
│   ├── index.tsx        ← Home (вкладка)
│   └── explore.tsx      ← Explore (вкладка)
├── product/
│   └── [id].tsx          ← екран деталей товару (НЕ вкладка)
└── _layout.tsx
```

Кореневий `_layout.tsx` зазвичай визначає `Stack`:

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="product/[id]" options={{ title: 'Деталі товару' }} />
    </Stack>
  );
}
```

Такий екран автоматично отримує системну шапку зі стрілкою "Назад" — це вбудована поведінка Stack-навігатора, додаткового коду не потрібно.

---

## Динамічні маршрути

Файл із назвою в квадратних дужках — `[id].tsx` — означає **динамічний сегмент** маршруту: замість `id` у реальному URL підставляється будь-яке значення.

```
app/product/[id].tsx   →   /product/1, /product/42, /product/abc — усі ведуть на цей файл
```

```tsx
// app/product/[id].tsx
import { useLocalSearchParams } from 'expo-router';
import { View, Text } from 'react-native';

export default function ProductScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <View>
      <Text>Товар з ID: {id}</Text>
    </View>
  );
}
```

---

## Передача даних через route-параметри

Найпоширеніший сценарій — список → перехід на деталі конкретного елемента.

```tsx
// список
import { router } from 'expo-router';

<Pressable onPress={() => router.push(`/product/${item.id}`)}>
  <Text>{item.name}</Text>
</Pressable>
```

```tsx
// екран деталей — отримує id і сам шукає повні дані
import { useLocalSearchParams } from 'expo-router';

export default function ProductScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const product = products.find((p) => p.id === id);

  return <Text>{product?.name}</Text>;
}
```

> **Головне правило:** через route-параметри передавай лише **ідентифікатор** (рядок/число), а не готовий об'єкт. Повні дані на екрані деталей знаходяться за цим ID — з локального масиву, глобального стану чи запиту до API.

---

## Query-параметри для кількох значень

Коли потрібно передати кілька простих значень одразу (наприклад, параметри фільтра чи пошуку):

```tsx
router.push({
  pathname: '/search',
  params: { query: 'взуття', category: 'спорт' },
});
```

```tsx
// екран пошуку
const { query, category } = useLocalSearchParams<{ query: string; category: string }>();
```

Це логічне продовження того самого підходу, що й з `[id]` — тільки замість одного динамічного сегмента передається набір іменованих значень, схожих на `?query=...&category=...` у веб-URL.

---

## Чому не варто передавати весь об'єкт напряму

Технічно можна серіалізувати об'єкт у JSON-рядок:

```tsx
// ❌ Так можна, але це погана практика
router.push({
  pathname: '/product/details',
  params: { data: JSON.stringify(product) },
});
```

Проблеми такого підходу:

- параметри маршруту — це рядки, тому доводиться постійно робити `JSON.stringify` / `JSON.parse`;
- дані "застигають" на момент переходу: якщо об'єкт зміниться (наприклад, оновиться ціна товару), екран деталей про це не дізнається, доки не відкриється заново;
- незручно й ненадійно для великих об'єктів чи масивів.

**Правильний підхід:** передавати ID, а сам об'єкт брати з джерела даних (масив у пам'яті, глобальний стан або запит на сервер) — це й дозволяє даним лишатися актуальними.

---

## Глобальний стан через Context API

Якщо дані потрібні одразу на кількох екранах (кошик покупок, дані користувача, список завдань) — правильніше підняти стан вище, а не "проштовхувати" його через route-параметри.

Уявімо структуру компонентів:
```
App
 └── Screen
      └── Header
           └── UserAvatar
```

Якщо App знає ім'я користувача (у своєму useState), а показати його треба в UserAvatar, яка на 3 рівні глибше — доводиться передавати name як props через кожен проміжний компонент, навіть якщо Screen і Header цей name взагалі не використовують, а лише "пропускають далі".

```tsx
function App() {
  const [name, setName] = useState('Олена');
  return <Screen name={name} />;
}

function Screen({ name }) {
  return <Header name={name} />; // сам не використовує, лише передає
}

function Header({ name }) {
  return <UserAvatar name={name} />; // теж лише передає
}

function UserAvatar({ name }) {
  return <Text>{name}</Text>; // тут нарешті використовується
}
```

Це називається **prop drilling** ("протягування пропсів") — і чим глибша структура компонентів (а в реальних застосунках вона глибша, ніж у прикладі), тим це стає незручнішим і крихкішим: додати новий проміжний компонент — означає знову протягувати через нього всі пропси.

**Context API** вирішує це так: дані кладуться в "спільний простір", і будь-який компонент нижче по дереву може взяти їх напряму, без участі проміжних компонентів.

### Як працювати з ContextAPI

#### Крок 1 — створити Context

```tsx
import { createContext } from 'react';

const UserContext = createContext(null);
```

`createContext(defaultValue)` створює об'єкт-контейнер. `defaultValue` (тут `null`) — значення, яке використовується, якщо компонент опиниться поза `Provider` (рідкісний випадок, зазвичай сигналізує про помилку в структурі).

#### Крок 2 — обгорнути дерево компонентів у Provider

```tsx
function App() {
  const [name, setName] = useState('Олена');

  return (
    <UserContext.Provider value={name}>
      <Screen />
    </UserContext.Provider>
  );
}
```

`Provider` — це спеціальний компонент, який автоматично створюється разом із `Context`. Усе, що знаходиться всередині нього (на будь-якій глибині вкладеності), отримує доступ до значення в `value`.

Зверни увагу: `Screen` тепер не приймає `name` як prop взагалі — він просто рендерить `Header` без жодних параметрів.

#### Крок 3 — прочитати значення через useContext

```tsx
import { useContext } from 'react';

function UserAvatar() {
  const name = useContext(UserContext);
  return <Text>{name}</Text>;
}
```

`UserAvatar` бере значення напряму з `UserContext`, навіть попри те, що `Screen` і `Header` між ними нічого про це не знають. Це і є головна перевага — розрив ланцюжка "протягування" пропсів.

### Повний приклад: кошик інтернет-магазину
```tsx
// contexts/CartContext.tsx
import { createContext, useContext, useState, ReactNode } from 'react';

type CartItem = { id: string; name: string; price: number };
type CartContextType = {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
};

const CartContext = createContext<CartContextType | undefined>(undefined);

export function CartProvider({ children }: { children: ReactNode }) {
  const [items, setItems] = useState<CartItem[]>([]);

  const addItem = (item: CartItem) => setItems((prev) => [...prev, item]);
  const removeItem = (id: string) =>
    setItems((prev) => prev.filter((i) => i.id !== id));

  return (
    <CartContext.Provider value={{ items, addItem, removeItem }}>
      {children}
    </CartContext.Provider>
  );
}

export function useCart() {
  const context = useContext(CartContext);
  if (!context) throw new Error('useCart має використовуватись усередині CartProvider');
  return context;
}
```

```tsx
// app/_layout.tsx — обгортаємо весь застосунок провайдером
import { CartProvider } from '../contexts/CartContext';
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <CartProvider>
      <Stack />
    </CartProvider>
  );
}
```

Тепер будь-який екран отримує доступ до тих самих даних одним рядком, без параметрів навігації взагалі:

```tsx
// на будь-якому екрані
import { useCart } from '../contexts/CartContext';

const { items, addItem } = useCart();
```

> Для більших проєктів, коли Context починає "тягнути" забагато (складна логіка, часті оновлення), замість нього часто беруть **Zustand** — бібліотеку зі схожою ідеєю, але менш багатослівну. Можна лишити як тему для наступних занять.

### Патерн "custom hook"

Замість того, щоб у кожному компоненті писати useContext(CounterContext) і перевіряти на null, прийнято робити свій хук:
```tsx
function useCounter() {
  const context = useContext(CounterContext);
  if (!context) {
    throw new Error('useCounter має використовуватись усередині CounterProvider');
  }
  return context;
}
```

Тепер використання виглядає чистіше:

```tsx
function CounterDisplay() {
  const { count } = useCounter(); // без useContext і перевірок вручну
  return <Text>Значення: {count}</Text>;
}
```

Ще одна перевага — якщо хтось випадково використає `useCounter()` поза `CounterProvider`, одразу отримає зрозумілу помилку замість тихого `undefined`, з яким важко зрозуміти, що пішло не так.

---

## 8. Локальне збереження: AsyncStorage

Context вирішує обмін даними між екранами **під час роботи застосунку**, але дані зникають при повному закритті. Щоб інформація лишалася й після перезапуску — потрібне локальне сховище.

### Встановлення

```bash
npm install @react-native-async-storage/async-storage@2.2.0
```

> В цьому матеріалі розглядається робота з версією **Async Storage версії 2.2.0**, яка сумісна з **React Native SDK 54**.
>
> Для більш нових версій React Native потрібно розглядати версію Async Storage 3+. Деталі можна знайти у документації:
> https://react-native-async-storage.github.io/latest/

### Базові операції

`AsyncStorage` зберігає лише рядки, тому об'єкти/масиви треба серіалізувати через `JSON.stringify`/`JSON.parse`. Усі методи — асинхронні (повертають `Promise`).

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// збереження
const saveCart = async (items: CartItem[]) => {
  await AsyncStorage.setItem('cart', JSON.stringify(items));
};

// зчитування
const loadCart = async (): Promise<CartItem[]> => {
  const raw = await AsyncStorage.getItem('cart');
  return raw ? JSON.parse(raw) : [];
};

// видалення
await AsyncStorage.removeItem('cart');

// повне очищення сховища
await AsyncStorage.clear();
```

### Типовий патерн: завантаження при старті екрана

```tsx
import { useState, useEffect } from 'react';
import AsyncStorage from '@react-native-async-storage/async-storage';

export function useCartWithStorage() {
  const [items, setItems] = useState<CartItem[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  // завантажити збережені дані один раз при монтуванні
  useEffect(() => {
    AsyncStorage.getItem('cart').then((raw) => {
      if (raw) setItems(JSON.parse(raw));
      setIsLoading(false);
    });
  }, []);

  // зберігати автоматично при кожній зміні
  useEffect(() => {
    if (!isLoading) {
      AsyncStorage.setItem('cart', JSON.stringify(items));
    }
  }, [items]);

  return { items, setItems, isLoading };
}
```

Це поєднує вже відомий `useEffect` (завантаження даних при монтуванні — та сама логіка, що й для запитів до API) із записом при кожній зміні стану.

---

## 9. Як обрати правильний підхід

| Ситуація | Рішення |
|---|---|
| Один екран → інший, просте значення (ID, фільтр) | Route/query-параметри |
| Дані потрібні одночасно на кількох екранах | Context API (пізніше — Zustand) |
| Дані мають лишатися після перезапуску застосунку | AsyncStorage |
| Дані з сервера, актуальні саме на момент відкриття екрана | Запит за ID через `useEffect`/`fetch` на самому екрані деталей |

На практиці ці підходи зазвичай комбінуються: Context тримає дані в пам'яті поки застосунок відкритий, а AsyncStorage синхронізує їх із диском для наступного запуску.

---

## 10. Комплексний приклад: список → деталі → збереження

Приклад об'єднує все заняття: список товарів із переходом на деталі за ID, кошик через Context, і збереження кошика в AsyncStorage.

```tsx
// app/(tabs)/index.tsx — список товарів
import { View, FlatList, Text, Pressable, StyleSheet } from 'react-native';
import { router } from 'expo-router';

const PRODUCTS = [
  { id: '1', name: 'Навушники', price: 1200 },
  { id: '2', name: 'Клавіатура', price: 2500 },
  { id: '3', name: 'Мишка', price: 800 },
];

export default function ProductListScreen() {
  return (
    <FlatList
      data={PRODUCTS}
      keyExtractor={(item) => item.id}
      contentContainerStyle={styles.list}
      renderItem={({ item }) => (
        <Pressable style={styles.row} onPress={() => router.push(`/product/${item.id}`)}>
          <Text style={styles.name}>{item.name}</Text>
          <Text style={styles.price}>{item.price} ₴</Text>
        </Pressable>
      )}
    />
  );
}

const styles = StyleSheet.create({
  list: { padding: 16 },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    padding: 14,
    borderBottomWidth: 1,
    borderBottomColor: '#e5e5ea',
  },
  name: { fontSize: 15 },
  price: { fontSize: 15, color: '#8e8e93' },
});
```

```tsx
// app/product/[id].tsx — деталі + додавання в кошик
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { useLocalSearchParams } from 'expo-router';
import { useCart } from '../../contexts/CartContext';

const PRODUCTS = [
  { id: '1', name: 'Навушники', price: 1200 },
  { id: '2', name: 'Клавіатура', price: 2500 },
  { id: '3', name: 'Мишка', price: 800 },
];

export default function ProductDetailsScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const { addItem } = useCart();
  const product = PRODUCTS.find((p) => p.id === id);

  if (!product) return <Text>Товар не знайдено</Text>;

  return (
    <View style={styles.container}>
      <Text style={styles.name}>{product.name}</Text>
      <Text style={styles.price}>{product.price} ₴</Text>
      <Pressable style={styles.button} onPress={() => addItem(product)}>
        <Text style={styles.buttonText}>Додати в кошик</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20 },
  name: { fontSize: 22, fontWeight: '700', marginBottom: 8 },
  price: { fontSize: 18, color: '#8e8e93', marginBottom: 20 },
  button: { backgroundColor: '#007AFF', padding: 14, borderRadius: 10, alignItems: 'center' },
  buttonText: { color: '#fff', fontWeight: '600' },
});
```

`CartProvider`, доповнений збереженням у `AsyncStorage` (які були розглянуті вище), дає кошик, що:
- доступний з будь-якого екрана без передачі параметрів;
- зберігається між перезапусками застосунку.
