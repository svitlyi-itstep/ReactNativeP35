# 4. Анімації в React Native: Animated API та Reanimated

## Зміст
1. [Вступ до анімацій у React Native](#вступ-до-анімацій-у-react-native)
2. [React Native Reanimated](#react-native-reanimated)
3. [Worklets та runOnJS](#worklets-та-runonjs)
4. [Reanimated 4: CSS-подібні анімації](#reanimated-4-css-подібні-анімації)
5. [Комплексний приклад: картка з лайком (Animated)](#комплексний-приклад-картка-з-лайком-animated)
6. [Комплексний приклад: перетягувана картка (Reanimated)](#комплексний-приклад-перетягувана-картка-reanimated)
7. [Корисні ресурси](#корисні-ресурси)

---

## Вступ до анімацій у React Native

У вебі браузер сам плавно інтерполює значення між станами через CSS `transition` і `@keyframes`. У React Native немає CSS — тому потрібен окремий механізм, який програє **проміжні кадри** між двома значеннями.

Ключова відмінність від того, що вже знайоме студентам (`useState` + умовний стиль): зміна `state` призводить до **миттєвої** зміни вигляду між рендерами. Анімація ж означає плавний перехід протягом певного часу.

Друга важлива особливість — **архітектура потоків**. React Native виконує логіку застосунку на JS-потоці, а рендеринг — на UI-потоці (нативному). Якщо анімація рахується на JS-потоці й той зайнятий (наприклад, обробляє великий список), анімація "смикається". Саме тому головне питання в анімаціях React Native — **де виконуються обчислення кожного кадру**.

У React Native є два підходи до анімування об'єктів: Animated та Reanimated:

| | Animated API | React Native Reanimated |
|---|---|---|
| Походження | вбудований у React Native | зовнішня бібліотека від Software Mansion |
| Встановлення | не потрібне | `npx expo install react-native-reanimated` |
| Де виконується анімація | JS-потік (за замовчуванням) або нативний (обмежено) | нативний UI-потік для всіх властивостей |
| Синтаксис | імперативний (`.start()`) | декларативний (хуки, shared values) |
| Жести | складно (`PanResponder`) | зручно (з `react-native-gesture-handler`) |
| Коли використовувати | навчання, прості анімації | реальні проєкти, складні/жестові анімації |

---

## Вбудований Animated API

### Базові поняття: Animated.Value та Animated-компоненти

Робота з `Animated` завжди складається з трьох кроків:

1. Створити анімоване значення — `Animated.Value`.
2. Запустити анімацію цього значення від одного числа до іншого.
3. Прив'язати значення до стилю через спеціальний анімований компонент.

```tsx
import { useRef } from 'react';
import { Animated } from 'react-native';

const fadeAnim = useRef(new Animated.Value(0)).current;
```

Без `useRef` значення `new Animated.Value(0)` пересоздаватиметься при кожному рендері компонента — і анімація "зламається". `useRef` гарантує, що об'єкт створюється один раз і живе весь час існування компонента.

Звичайний `<View>` не вміє приймати `Animated.Value` у стилі. Для цього є спеціальні версії:

- `Animated.View`
- `Animated.Text`
- `Animated.Image`
- `Animated.ScrollView`
- `Animated.FlatList`

Будь-який власний компонент можна зробити анімованим через `Animated.createAnimatedComponent(MyComponent)`.

### Animated.timing

Найпоширеніший вид анімації — перехід від одного значення до іншого за визначений час.

```tsx
import { useRef } from 'react';
import { View, Text, Pressable, Animated, Easing, StyleSheet } from 'react-native';

export default function FadeInExample() {
  const fadeAnim = useRef(new Animated.Value(0)).current;

  const showBlock = () => {
    Animated.timing(fadeAnim, {
      toValue: 1,
      duration: 500,
      easing: Easing.out(Easing.ease),
      useNativeDriver: true,
    }).start();
  };

  return (
    <View style={styles.container}>
      <Pressable style={styles.button} onPress={showBlock}>
        <Text style={styles.buttonText}>Показати</Text>
      </Pressable>

      <Animated.View style={[styles.box, { opacity: fadeAnim }]}>
        <Text>Привіт! Я плавно з'явився</Text>
      </Animated.View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 20, gap: 16 },
  button: { backgroundColor: '#007AFF', padding: 12, borderRadius: 8, alignItems: 'center' },
  buttonText: { color: '#fff', fontWeight: '600' },
  box: { backgroundColor: '#e5f1ff', padding: 20, borderRadius: 12 },
});
```

#### Параметри `Animated.timing`

| Параметр | Призначення |
|---|---|
| `toValue` | цільове значення |
| `duration` | тривалість у мс (за замовчуванням 500) |
| `easing` | функція згладжування швидкості |
| `delay` | затримка перед стартом у мс |
| `useNativeDriver` | виконання на нативному потоці (розділ 10) |

### Модуль `Easing`

Визначає "характер" руху — рівномірний, з прискоренням, з "відскоком":

```tsx
import { Easing } from 'react-native';

Easing.linear              // рівномірно
Easing.ease                // плавний старт і кінець
Easing.in(Easing.quad)     // повільний старт, швидкий кінець
Easing.out(Easing.quad)    // швидкий старт, повільний кінець
Easing.inOut(Easing.cubic) // повільно → швидко → повільно
Easing.bounce              // "відскок" наприкінці
Easing.elastic(1)          // пружинне "розгойдування"
```

### Колбек після завершення

```tsx
Animated.timing(fadeAnim, { ... }).start(({ finished }) => {
  if (finished) console.log('Анімація завершилась');
});
```

`finished` буде `false`, якщо анімацію перервали (наприклад, запустили нову до завершення попередньої).

### Animated.spring

Замість рівномірного руху — фізична імітація пружини з невеликим "перестрибуванням" і згасанням. Часто виглядає природніше для UI-елементів.

```tsx
const scaleAnim = useRef(new Animated.Value(1)).current;

const bounce = () => {
  Animated.spring(scaleAnim, {
    toValue: 1.2,
    friction: 3,    // тертя: менше = сильніше розгойдування
    tension: 40,    // натяг: більше = швидше
    useNativeDriver: true,
  }).start();
};

<Animated.View style={{ transform: [{ scale: scaleAnim }] }}>
  <Text>Пружинний елемент</Text>
</Animated.View>
```

Замість `friction`/`tension` можна використати `speed`/`bounciness` — простіші для інтуїтивного налаштування:

```tsx
Animated.spring(scaleAnim, {
  toValue: 1,
  speed: 20,        // швидкість (за замовчуванням 12)
  bounciness: 10,   // "стрибучість" (за замовчуванням 8)
  useNativeDriver: true,
});
```

### interpolate: перетворення значень і кольорів

`Animated.Value` — це просто число. `interpolate()` перетворює один діапазон значень у інший — числа, рядки з одиницями (`'180deg'`), кольори.

#### Числовий діапазон

```tsx
const moveAnim = useRef(new Animated.Value(0)).current;

const translateX = moveAnim.interpolate({
  inputRange: [0, 1],
  outputRange: [0, 200],
});

<Animated.View style={{ transform: [{ translateX }] }} />
```

#### Обертання (рядки з одиницями)

```tsx
const rotate = moveAnim.interpolate({
  inputRange: [0, 1],
  outputRange: ['0deg', '360deg'],
});

<Animated.View style={{ transform: [{ rotate }] }} />
```

#### Кілька опорних точок

```tsx
const opacity = scrollY.interpolate({
  inputRange: [0, 50, 100],
  outputRange: [1, 0.5, 0],
  extrapolate: 'clamp', // не виходити за межі outputRange
});
```

`extrapolate: 'clamp'` — важливий параметр: без нього при `inputRange` поза межами значення продовжать "екстраполюватись" (наприклад, `opacity` стане від'ємним).

#### Кольори

```tsx
const backgroundColor = animatedValue.interpolate({
  inputRange: [0, 0.5, 1],
  outputRange: ['#ff3b30', '#ffcc00', '#34c759'], // червоний → жовтий → зелений
});

<Animated.View style={{ backgroundColor }} />
```

> Анімація кольору у вбудованому `Animated` потребує `useNativeDriver: false` (розділ 10).

### Комбінатори: sequence, parallel, delay, loop

```tsx
// одна за одною
Animated.sequence([
  Animated.timing(scaleAnim, { toValue: 1.2, duration: 150, useNativeDriver: true }),
  Animated.timing(scaleAnim, { toValue: 1, duration: 150, useNativeDriver: true }),
]).start();

// одночасно
Animated.parallel([
  Animated.timing(fadeAnim, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(slideAnim, { toValue: 0, duration: 300, useNativeDriver: true }),
]).start();

// затримка між анімаціями в sequence
Animated.sequence([
  Animated.timing(fadeAnim, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.delay(1000),
  Animated.timing(fadeAnim, { toValue: 0, duration: 300, useNativeDriver: true }),
]).start();

// нескінченне повторення
Animated.loop(
  Animated.sequence([
    Animated.timing(pulseAnim, { toValue: 1.3, duration: 600, useNativeDriver: true }),
    Animated.timing(pulseAnim, { toValue: 1, duration: 600, useNativeDriver: true }),
  ])
).start();

// stagger: анімації одна за одною із зсувом, але з накладанням
Animated.stagger(100, [
  Animated.timing(item1Anim, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(item2Anim, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(item3Anim, { toValue: 1, duration: 300, useNativeDriver: true }),
]).start();
```

`Animated.stagger` корисний для "каскадної" появи елементів списку — кожен наступний стартує через 100 мс після попереднього.

#### Зупинка анімації

```tsx
const animation = Animated.loop(...);
animation.start();
// пізніше:
animation.stop();

// або через саме значення:
pulseAnim.stopAnimation();
```

Loop-анімації варто зупиняти при розмонтуванні компонента — у `useEffect` через cleanup-функцію:

```tsx
useEffect(() => {
  const animation = Animated.loop(...);
  animation.start();
  return () => animation.stop();
}, []);
```

### useNativeDriver: обмеження та правила

`useNativeDriver: true` переносить обчислення кадрів анімації з JS-потоку на нативний. Це дає значно плавнішу анімацію, особливо коли JS зайнятий.

**Обмеження:** нативний драйвер підтримує лише властивості, що не потребують перерахунку макета:

| `useNativeDriver: true` | `useNativeDriver: false` |
|---|---|
| `opacity` | `backgroundColor`, `color`, `borderColor` |
| `transform`: `scale`, `translateX/Y`, `rotate`, `skew` | `width`, `height` |
| | `top`, `left`, `right`, `bottom` |
| | `padding`, `margin` |
| | `borderRadius`, `borderWidth`, `fontSize` |

> Якщо є можливість замінити `width` на `scaleX`, `left` на `translateX` — варто це робити, щоб залишитись на нативному драйвері.

Якщо запустити непідтримувану властивість з `useNativeDriver: true` — React Native видасть помилку одразу, ще до старту анімації.

---

## React Native Reanimated

### Що таке Reanimated і чим він кращий

**React Native Reanimated** — бібліотека від Software Mansion, яка переосмислює анімації в React Native. Головні переваги над вбудованим `Animated`:

- **Усе на нативному потоці.** Будь-яка властивість — колір, розмір, відступи — анімується без обмежень `useNativeDriver`.
- **Декларативний синтаксис.** Замість `.start()` — хуки й "shared values", ближче до звичного React-стилю.
- **Worklets.** Функції, що виконуються прямо на UI-потоці, дозволяють писати складну логіку анімації (наприклад, реакцію на жест) без "пінг-понгу" між потоками.
- **Layout-анімації "з коробки".** `entering`/`exiting`/`layout` для появи, зникнення й перебудови елементів.
- **Інтеграція з жестами.** Разом із `react-native-gesture-handler` — стандарт для драг-н-дропу, свайпів, pinch-to-zoom.

Актуальна версія — **Reanimated 4**. Вона працює лише з новою архітектурою React Native (яка в Expo SDK 5x увімкнена за замовчуванням) і винесла механізм worklets в окремий пакет `react-native-worklets`.

### Встановлення в Expo-проєкт

```bash
npx expo install react-native-reanimated react-native-worklets
```

Для жестів — додатково:

```bash
npx expo install react-native-gesture-handler
```

У сучасних Expo-проєктах `babel-preset-expo` автоматично підключає потрібний Babel-плагін, тому додаткова конфігурація `babel.config.js` зазвичай не потрібна. Якщо ж проєкт має власний `babel.config.js`, плагін має бути **останнім** у списку:

```js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: ['react-native-worklets/plugin'],
};
```

Після встановлення варто перезапустити Metro з очищенням кешу:

```bash
npx expo start --clear
```

> Багато проєктів, створених через `create-expo-app`, уже мають Reanimated у залежностях, бо він потрібен для навігації. Варто перевірити `package.json`, перш ніж встановлювати.

### Базові поняття: useSharedValue та useAnimatedStyle

Аналогія з `Animated`:

| Animated | Reanimated |
|---|---|
| `useRef(new Animated.Value(0)).current` | `useSharedValue(0)` |
| `Animated.timing(value, {...}).start()` | `value.value = withTiming(1)` |
| `style={{ opacity: fadeAnim }}` | `useAnimatedStyle(() => ({ opacity: opacity.value }))` |
| `Animated.View` | `Animated.View` (з `react-native-reanimated`) |

```tsx
import { View, Pressable, Text, StyleSheet } from 'react-native';
import Animated, { useSharedValue, useAnimatedStyle, withTiming } from 'react-native-reanimated';

export default function FadeInReanimated() {
  const opacity = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
  }));

  const show = () => {
    opacity.value = withTiming(1, { duration: 500 });
  };

  return (
    <View style={styles.container}>
      <Pressable style={styles.button} onPress={show}>
        <Text style={styles.buttonText}>Показати</Text>
      </Pressable>
      <Animated.View style={[styles.box, animatedStyle]}>
        <Text>Привіт від Reanimated</Text>
      </Animated.View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 20, gap: 16 },
  button: { backgroundColor: '#007AFF', padding: 12, borderRadius: 8, alignItems: 'center' },
  buttonText: { color: '#fff', fontWeight: '600' },
  box: { backgroundColor: '#e5f1ff', padding: 20, borderRadius: 12 },
});
```

#### Ключові моменти

- **`useSharedValue(0)`** створює значення, "спільне" між JS- та UI-потоками. Читання й запис — через `.value`.
- **`useAnimatedStyle`** — функція, яка перераховується на UI-потоці щоразу, коли змінюється будь-яке shared value, використане всередині.
- **Присвоєння `opacity.value = withTiming(1)`** — саме так запускається анімація. Не `.start()`, а просто присвоєння "анімованого" значення.
- Змінювати `.value` **можна прямо в обробнику** — без `useEffect` чи додаткових хуків.

#### Функції анімації: withTiming, withSpring, withRepeat

```tsx
import { withTiming, withSpring, withRepeat, Easing } from 'react-native-reanimated';

// плавний перехід за час
opacity.value = withTiming(1, {
  duration: 500,
  easing: Easing.out(Easing.exp),
});

// пружина
scale.value = withSpring(1.2, {
  damping: 10,     // затухання (аналог friction)
  stiffness: 100,  // жорсткість (аналог tension)
});

// повторення
rotation.value = withRepeat(
  withTiming(360, { duration: 1000, easing: Easing.linear }),
  -1,     // кількість повторів: -1 = нескінченно
  false   // reverse: чи повертатись назад після кожного циклу
);
```

#### Колбек після завершення

```tsx
opacity.value = withTiming(0, { duration: 300 }, (finished) => {
  if (finished) {
    // виконується на UI-потоці! Для виклику JS-коду — runOnJS (розділ 20)
  }
});
```

### Композиція: withSequence, withDelay

```tsx
import { withSequence, withDelay, withTiming, withSpring } from 'react-native-reanimated';

// послідовність: стрибок і повернення
scale.value = withSequence(
  withSpring(1.3),
  withSpring(1)
);

// затримка перед анімацією
opacity.value = withDelay(500, withTiming(1, { duration: 300 }));

// комбінація: shake-ефект
translateX.value = withSequence(
  withTiming(-10, { duration: 50 }),
  withTiming(10, { duration: 50 }),
  withTiming(-10, { duration: 50 }),
  withTiming(0, { duration: 50 })
);
```

Аналог `Animated.parallel` не потрібен — достатньо присвоїти кілька shared values одночасно:

```tsx
const appear = () => {
  opacity.value = withTiming(1);
  translateY.value = withSpring(0);
};
```

### interpolate та interpolateColor у Reanimated

```tsx
import { interpolate, interpolateColor, Extrapolation } from 'react-native-reanimated';

const animatedStyle = useAnimatedStyle(() => {
  const translateY = interpolate(
    progress.value,
    [0, 1],
    [50, 0],
    Extrapolation.CLAMP
  );

  const backgroundColor = interpolateColor(
    progress.value,
    [0, 1],
    ['#007AFF', '#34C759']
  );

  return {
    transform: [{ translateY }],
    backgroundColor,
  };
});
```

Головна відмінність від `Animated`: **`interpolateColor` працює повністю на UI-потоці**, без жодних обмежень — на відміну від вбудованого API, де для кольору потрібен `useNativeDriver: false`.

#### Приклад: кнопка з плавною зміною кольору

```tsx
import Animated, { useSharedValue, useAnimatedStyle, withTiming, interpolateColor } from 'react-native-reanimated';
import { Pressable, Text, StyleSheet } from 'react-native';

export default function ColorButton() {
  const progress = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    backgroundColor: interpolateColor(progress.value, [0, 1], ['#007AFF', '#34C759']),
  }));

  return (
    <Pressable
      onPressIn={() => (progress.value = withTiming(1, { duration: 200 }))}
      onPressOut={() => (progress.value = withTiming(0, { duration: 200 }))}
    >
      <Animated.View style={[styles.button, animatedStyle]}>
        <Text style={styles.text}>Натисни й тримай</Text>
      </Animated.View>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  button: { padding: 14, borderRadius: 10, alignItems: 'center' },
  text: { color: '#fff', fontWeight: '600' },
});
```

### Layout-анімації: entering / exiting

Одна з найзручніших можливостей Reanimated — готові анімації появи, зникнення та перебудови елементів без жодних shared values:

```tsx
import Animated, { FadeIn, FadeOut, SlideInRight, SlideOutLeft, Layout } from 'react-native-reanimated';

<Animated.View
  entering={FadeIn.duration(300)}
  exiting={FadeOut.duration(200)}
  layout={Layout.springify()}
>
  <Text>Цей елемент плавно з'являється і зникає</Text>
</Animated.View>
```

#### Готові анімації

| Категорія | Приклади |
|---|---|
| Fade | `FadeIn`, `FadeOut`, `FadeInUp`, `FadeInDown` |
| Slide | `SlideInRight`, `SlideInLeft`, `SlideOutDown`, `SlideInUp` |
| Zoom | `ZoomIn`, `ZoomOut`, `ZoomInRotate` |
| Bounce | `BounceIn`, `BounceOut` |
| Flip | `FlipInXUp`, `FlipOutYLeft` |

Кожну можна налаштувати ланцюжком: `.duration(500)`, `.delay(200)`, `.springify()`, `.damping(10)`.

#### Приклад: список з анімованим додаванням/видаленням

```tsx
import { useState } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';
import Animated, { FadeInDown, FadeOutLeft, LinearTransition } from 'react-native-reanimated';

export default function AnimatedTaskList() {
  const [tasks, setTasks] = useState(['Купити хліб', 'Зателефонувати мамі', 'Прибрати']);

  const addTask = () => setTasks((prev) => [`Завдання ${prev.length + 1}`, ...prev]);
  const removeTask = (task: string) => setTasks((prev) => prev.filter((t) => t !== task));

  return (
    <View style={styles.container}>
      <Pressable style={styles.addButton} onPress={addTask}>
        <Text style={styles.addText}>+ Додати</Text>
      </Pressable>

      {tasks.map((task) => (
        <Animated.View
          key={task}
          entering={FadeInDown.duration(300)}
          exiting={FadeOutLeft.duration(200)}
          layout={LinearTransition.springify()}
          style={styles.row}
        >
          <Text style={styles.rowText}>{task}</Text>
          <Pressable onPress={() => removeTask(task)}>
            <Text style={styles.delete}>Видалити</Text>
          </Pressable>
        </Animated.View>
      ))}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 20, gap: 8 },
  addButton: { backgroundColor: '#007AFF', padding: 12, borderRadius: 8, alignItems: 'center', marginBottom: 8 },
  addText: { color: '#fff', fontWeight: '600' },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 14,
    backgroundColor: '#f2f2f7',
    borderRadius: 8,
  },
  rowText: { fontSize: 15 },
  delete: { color: '#ff3b30', fontWeight: '600' },
});
```

`layout={LinearTransition.springify()}` — анімує **зсув інших елементів**, коли один із них додається чи видаляється. Це прямий аналог `LayoutAnimation` з вбудованого API, але без Android-обмежень і з більшою гнучкістю.

## Worklets та runOnJS

**Worklet** — це функція, яка виконується на UI-потоці. Reanimated автоматично перетворює на worklets функції всередині `useAnimatedStyle`, обробники жестів, колбеки `withTiming` тощо.

Наслідок: **із worklet не можна напряму викликати звичайний JS-код** — `setState`, навігацію, `console.log` з об'єктами тощо. Для цього є `runOnJS`:

```tsx
import { runOnJS } from 'react-native-reanimated';

const [isDone, setIsDone] = useState(false);

const markDone = () => setIsDone(true);

opacity.value = withTiming(0, { duration: 300 }, (finished) => {
  if (finished) {
    runOnJS(markDone)(); // виклик JS-функції з UI-потоку
  }
});
```

```tsx
// приклад із жестом: навігація після свайпу
const goBack = () => router.back();

const swipeGesture = Gesture.Pan().onEnd((event) => {
  if (event.translationX > 100) {
    runOnJS(goBack)();
  }
});
```

Без `runOnJS` виклик `setIsDone(true)` всередині worklet призведе до помилки або тихого ігнорування.

## Reanimated 4: CSS-подібні анімації

Reanimated 4 додав API, що імітує CSS `transition` та `@keyframes` — для простих випадків він ще коротший за shared values:

### CSS Transitions

```tsx
import Animated from 'react-native-reanimated';
import { useState } from 'react';

function ToggleBox() {
  const [expanded, setExpanded] = useState(false);

  return (
    <Animated.View
      style={{
        width: expanded ? 200 : 100,
        height: 100,
        backgroundColor: expanded ? '#34C759' : '#007AFF',
        transitionProperty: ['width', 'backgroundColor'],
        transitionDuration: 300,
        transitionTimingFunction: 'ease-in-out',
      }}
      onTouchEnd={() => setExpanded(!expanded)}
    />
  );
}
```

Зміна `width` через звичайний `useState` автоматично анімується завдяки `transitionProperty` — без shared values узагалі.

### CSS Animations (keyframes)

```tsx
<Animated.View
  style={{
    width: 60,
    height: 60,
    backgroundColor: '#ff3b30',
    animationName: {
      from: { transform: [{ rotate: '0deg' }] },
      to: { transform: [{ rotate: '360deg' }] },
    },
    animationDuration: 1000,
    animationIterationCount: 'infinite',
    animationTimingFunction: 'linear',
  }}
/>
```

Цей підхід зручний для тих, хто прийшов із вебу — синтаксис майже ідентичний CSS. Для складніших сценаріїв (жести, взаємозалежні анімації) shared values лишаються основним інструментом.

---

## Комплексний приклад: картка з лайком (Animated)

```tsx
import { useEffect, useRef, useState } from 'react';
import { Text, Pressable, Animated, StyleSheet } from 'react-native';
import { Ionicons } from '@expo/vector-icons';

export default function AnimatedLikeCard() {
  const fadeAnim = useRef(new Animated.Value(0)).current;
  const likeScale = useRef(new Animated.Value(1)).current;
  const colorAnim = useRef(new Animated.Value(0)).current;
  const [isLiked, setIsLiked] = useState(false);

  useEffect(() => {
    Animated.timing(fadeAnim, { toValue: 1, duration: 400, useNativeDriver: true }).start();
  }, []);

  const handleLike = () => {
    setIsLiked((prev) => !prev);

    Animated.sequence([
      Animated.spring(likeScale, { toValue: 1.4, useNativeDriver: true, friction: 3 }),
      Animated.spring(likeScale, { toValue: 1, useNativeDriver: true, friction: 3 }),
    ]).start();

    Animated.timing(colorAnim, {
      toValue: isLiked ? 0 : 1,
      duration: 300,
      useNativeDriver: false,
    }).start();
  };

  const backgroundColor = colorAnim.interpolate({
    inputRange: [0, 1],
    outputRange: ['#f2f2f7', '#ffe5e5'],
  });

  return (
    <Animated.View style={[styles.card, { opacity: fadeAnim }]}>
      <Text style={styles.title}>Захід вихідного дня</Text>
      <Text style={styles.subtitle}>Прогулянка набережною о 18:00</Text>

      <Pressable onPress={handleLike}>
        <Animated.View style={[styles.likeButton, { backgroundColor }]}>
          <Animated.View style={{ transform: [{ scale: likeScale }] }}>
            <Ionicons
              name={isLiked ? 'heart' : 'heart-outline'}
              size={22}
              color={isLiked ? '#ff3b30' : '#8e8e93'}
            />
          </Animated.View>
          <Text style={styles.likeText}>{isLiked ? 'Подобається' : 'Лайк'}</Text>
        </Animated.View>
      </Pressable>
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  card: {
    margin: 20,
    padding: 20,
    borderRadius: 16,
    backgroundColor: '#fff',
    gap: 8,
    shadowColor: '#000',
    shadowOpacity: 0.08,
    shadowRadius: 8,
    elevation: 2,
  },
  title: { fontSize: 17, fontWeight: '700' },
  subtitle: { fontSize: 14, color: '#8e8e93', marginBottom: 8 },
  likeButton: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
    alignSelf: 'flex-start',
    paddingVertical: 8,
    paddingHorizontal: 14,
    borderRadius: 20,
  },
  likeText: { fontSize: 14, fontWeight: '600', color: '#3c3c43' },
});
```

---

## Комплексний приклад: перетягувана картка (Reanimated)

Та сама картка, але на Reanimated — з жестом перетягування, поверненням на місце та "стрибком" лайка. Демонструє, наскільки коротшим стає код.

```tsx
import { useState } from 'react';
import { Text, Pressable, StyleSheet } from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withSequence,
  withTiming,
  interpolateColor,
  FadeInUp,
} from 'react-native-reanimated';

export default function ReanimatedLikeCard() {
  const [isLiked, setIsLiked] = useState(false);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const likeScale = useSharedValue(1);
  const likeProgress = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((e) => {
      translateX.value = e.translationX;
      translateY.value = e.translationY;
    })
    .onEnd(() => {
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
    });

  const cardStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }, { translateY: translateY.value }],
  }));

  const likeButtonStyle = useAnimatedStyle(() => ({
    backgroundColor: interpolateColor(likeProgress.value, [0, 1], ['#f2f2f7', '#ffe5e5']),
  }));

  const likeIconStyle = useAnimatedStyle(() => ({
    transform: [{ scale: likeScale.value }],
  }));

  const handleLike = () => {
    const next = !isLiked;
    setIsLiked(next);
    likeScale.value = withSequence(withSpring(1.4), withSpring(1));
    likeProgress.value = withTiming(next ? 1 : 0, { duration: 300 });
  };

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View entering={FadeInUp.duration(400)} style={[styles.card, cardStyle]}>
        <Text style={styles.title}>Захід вихідного дня</Text>
        <Text style={styles.subtitle}>Перетягни картку — вона повернеться</Text>

        <Pressable onPress={handleLike}>
          <Animated.View style={[styles.likeButton, likeButtonStyle]}>
            <Animated.View style={likeIconStyle}>
              <Ionicons
                name={isLiked ? 'heart' : 'heart-outline'}
                size={22}
                color={isLiked ? '#ff3b30' : '#8e8e93'}
              />
            </Animated.View>
            <Text style={styles.likeText}>{isLiked ? 'Подобається' : 'Лайк'}</Text>
          </Animated.View>
        </Pressable>
      </Animated.View>
    </GestureDetector>
  );
}

const styles = StyleSheet.create({
  card: {
    margin: 20,
    padding: 20,
    borderRadius: 16,
    backgroundColor: '#fff',
    gap: 8,
    shadowColor: '#000',
    shadowOpacity: 0.08,
    shadowRadius: 8,
    elevation: 2,
  },
  title: { fontSize: 17, fontWeight: '700' },
  subtitle: { fontSize: 14, color: '#8e8e93', marginBottom: 8 },
  likeButton: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
    alignSelf: 'flex-start',
    paddingVertical: 8,
    paddingHorizontal: 14,
    borderRadius: 20,
  },
  likeText: { fontSize: 14, fontWeight: '600', color: '#3c3c43' },
});
```

> Для роботи цього прикладу `GestureHandlerRootView` має обгортати застосунок у кореневому `_layout.tsx`.

---

## Корисні ресурси

### Офіційна документація

- [Animations — React Native Docs](https://reactnative.dev/docs/animations) — загальний огляд анімацій у React Native, з якого варто почати.
- [Animated API Reference](https://reactnative.dev/docs/animated) — повний довідник вбудованого `Animated`.
- [Easing](https://reactnative.dev/docs/easing) — усі доступні функції згладжування з візуалізацією.
- [LayoutAnimation](https://reactnative.dev/docs/layoutanimation) — документація вбудованих layout-анімацій.
- [React Native Reanimated Docs](https://docs.swmansion.com/react-native-reanimated/) — головна сторінка документації Reanimated з розділами Fundamentals, Layout Animations, Guides.
- [React Native Gesture Handler Docs](https://docs.swmansion.com/react-native-gesture-handler/) — документація бібліотеки жестів.
- [Reanimated в Expo SDK](https://docs.expo.dev/versions/latest/sdk/reanimated/) — сторінка Expo про встановлення та особливості інтеграції.
- [Gesture Handler в Expo SDK](https://docs.expo.dev/versions/latest/sdk/gesture-handler/) — аналогічна сторінка для жестів.

### Репозиторії

- [software-mansion/react-native-reanimated](https://github.com/software-mansion/react-native-reanimated) — офіційний репозиторій; у папці `apps/common-app` є шоукейс із десятками прикладів анімацій.
- [software-mansion/react-native-gesture-handler](https://github.com/software-mansion/react-native-gesture-handler) — репозиторій Gesture Handler з прикладами.

### Спільнота

- [Software Mansion Community Discord](https://discord.swmansion.com) — офіційний Discord авторів Reanimated та Gesture Handler для питань.
- [Expo Discord](https://chat.expo.dev) — спільнота Expo, де часто обговорюють інтеграцію анімацій у Expo-проєкти.