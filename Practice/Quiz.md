# Вікторина з таймером


Створити застосунок-вікторину з кількома питаннями, де на кожне питання відведено обмежений час.

## Структура застосунку

```
app/
├── (tabs)/
│   └── index.tsx        ← стартовий екран
├── quiz.tsx             ← екран з питаннями
├── result.tsx           ← екран результатів
└── _layout.tsx
data/
└── questions.ts         ← масив питань
```

Питання зберігаються в окремому файлі як масив об'єктів:

```ts
export const QUESTIONS = [
  {
    id: '1',
    question: 'Який компонент у React Native замінює <div>?',
    options: ['Text', 'View', 'Container', 'Box'],
    correctIndex: 1,
  },
  // ... щонайменше 5 питань
];
```

---

## Екран 1: Стартовий

- Назва вікторини, короткий опис (кількість питань, час на відповідь).
- Кнопка "Почати", яка при появі екрана плавно "пульсує" (`Animated.loop`), привертаючи увагу.
- Натискання веде на екран вікторини через `router.push('/quiz')`.

---

## Екран 2: Вікторина

### Функціонал

- Зверху: номер питання ("Питання 3 з 5") та поточний рахунок.
- Смужка таймера: на кожне питання відводиться 15 секунд.
- Текст питання та 4 кнопки з варіантами відповідей.
- Після вибору відповіді кнопки блокуються, через 1 секунду відбувається перехід до наступного питання.
- Якщо час вийшов, а відповіді немає, питання зараховується як неправильне, і вікторина переходить далі.
- Кнопка "Вийти" в шапці: `Alert` із підтвердженням ("Прогрес буде втрачено. Вийти?"), при підтвердженні `router.back()`.
- Після останнього питання перехід на екран результатів через `router.replace()` з передачею параметрів `score` і `total`, щоб кнопка "Назад" не повертала в пройдену вікторину.

### Анімації

**Смужка таймера.** Ширина плавно зменшується від 100% до 0% за 15 секунд (`Animated.timing` з `Easing.linear`). Колір одночасно змінюється від зеленого через жовтий до червоного (`interpolate` з трьома точками). Обидві властивості потребують `useNativeDriver: false`.

**Правильна відповідь.** Кнопка плавно зафарбовується в зелений колір і робить короткий "стрибок" (`sequence` з двох `spring`).

**Неправильна відповідь.** Кнопка червоніє й "трясеться" (`sequence` з кількох коротких `timing` для `translateX`: 0 → -10 → 10 → -10 → 0). Правильний варіант при цьому теж підсвічується зеленим, щоб користувач побачив, яка відповідь була вірною.

**Поява питання.** Кожне нове питання з'являється з анімацією: текст і кнопки проявляються та злегка "заїжджають" знизу (`parallel`: `opacity` + `translateY`).

### Стан екрана

```ts
const [currentIndex, setCurrentIndex] = useState(0);
const [score, setScore] = useState(0);
const [selectedIndex, setSelectedIndex] = useState<number | null>(null);
const [isLocked, setIsLocked] = useState(false);
```

---

## Екран 3: Результати

- Отримує `score` і `total` через `useLocalSearchParams` (параметри приходять рядками, тому потрібен `Number()`).
- Велике число балів "набігає" від 0 до фінального результату (`Animated.Value` + `addListener` для оновлення тексту).
- Колір і повідомлення залежать від результату:
  - понад 80% — "Чудово!";
  - 50–80% — "Непогано";
  - менше 50% — "Варто повторити матеріал".
- Кнопка "Пройти ще раз" веде знову на `/quiz` через `router.replace()`.
- Кнопка "На головну" повертає на стартовий екран.

---

## Підказки щодо реалізації

### Запуск таймера на кожне питання

Скидати значення й запускати анімацію варто в `useEffect`, що залежить від `currentIndex`:

```ts
useEffect(() => {
  timerAnim.setValue(1);
  const animation = Animated.timing(timerAnim, {
    toValue: 0,
    duration: 15000,
    easing: Easing.linear,
    useNativeDriver: false,
  });
  animation.start(({ finished }) => {
    if (finished) handleTimeOut();
  });
  return () => animation.stop();
}, [currentIndex]);
```

Ключовий момент тут `finished` у колбеку: він дорівнює `true`, лише якщо анімація дійшла до кінця сама. Якщо користувач відповів раніше й анімацію зупинили через `stop()`, колбек отримає `finished: false`, і питання не буде помилково зараховане як "час вийшов".

### Зупинка таймера при відповіді

Після вибору варіанта таймер треба зупинити (`timerAnim.stopAnimation()`), інакше він продовжить рахувати під час паузи перед наступним питанням.

### Ширина у відсотках

Для прив'язки ширини смужки зручно інтерполювати значення в рядок:

```ts
const width = timerAnim.interpolate({
  inputRange: [0, 1],
  outputRange: ['0%', '100%'],
});

const backgroundColor = timerAnim.interpolate({
  inputRange: [0, 0.5, 1],
  outputRange: ['#ff3b30', '#ffcc00', '#34c759'],
});
```

### Окреме анімоване значення для кожної кнопки

Для "трясіння" й "стрибка" кожна кнопка-варіант потребує власного `Animated.Value`. Найпростіше винести кнопку в окремий компонент `AnswerButton`, який усередині має свої `useRef(new Animated.Value(...))` і запускає потрібну анімацію залежно від пропсів (`isCorrect`, `isSelected`, `isLocked`).

### "Набігання" числа на екрані результатів

```ts
const countAnim = useRef(new Animated.Value(0)).current;
const [displayScore, setDisplayScore] = useState(0);

useEffect(() => {
  const id = countAnim.addListener(({ value }) => setDisplayScore(Math.round(value)));
  Animated.timing(countAnim, {
    toValue: Number(score),
    duration: 1000,
    useNativeDriver: false,
  }).start();
  return () => countAnim.removeListener(id);
}, []);
```

## Додаткові завдання

- Перемішувати порядок питань і варіантів відповідей при кожному запуску.
- Зберігати найкращий результат в `AsyncStorage` і показувати його на стартовому екрані.
- В останні 5 секунд смужка таймера починає пульсувати, щоб створити відчуття поспіху.
- Додати вибір категорії питань на стартовому екрані з передачею її в `quiz` через параметри.

---

## Корисні посилання

- [Animated — React Native Docs](https://reactnative.dev/docs/animated)
- [Easing — React Native Docs](https://reactnative.dev/docs/easing)
- [Alert — React Native Docs](https://reactnative.dev/docs/alert)
- [Expo Router: навігація](https://docs.expo.dev/router/navigating-pages/)
- [Expo Router: параметри маршрутів](https://docs.expo.dev/router/reference/url-parameters/)
