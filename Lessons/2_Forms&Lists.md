# 2. Списки, форми та введення даних

## Зміст

1. [TextInput: базове введення тексту](#textinput-базове-введення-тексту)
2. [Типи клавіатур та спеціальні поля](#типи-клавіатур-та-спеціальні-поля)
3. [Багаторядкове поле та KeyboardAvoidingView](#багаторядкове-поле-та-keyboardavoidingview)
4. [Проста валідація введення](#проста-валідація-введення)
5. [FlatList: базовий синтаксис](#flatlist-базовий-синтаксис)
6. [Корисні властивості FlatList](#корисні-властивості-flatlist)
7. [SectionList: списки з групами](#sectionlist-списки-з-групами)
8. [Комплексний приклад: форма + список](#комплексний-приклад-форма--список)

---

## TextInput: базове введення тексту

У React Native немає тегу `<input>` — його аналог, **`TextInput`**, працює за принципом **контрольованого компонента**, який вже знайомий із React (web): значення зберігається в стані, а не в самому DOM-елементі.

```tsx
import { useState } from 'react';
import { View, TextInput, Text, StyleSheet } from 'react-native';

export default function SimpleInput() {
  const [name, setName] = useState('');

  return (
    <View style={styles.container}>
      <TextInput
        style={styles.input}
        placeholder="Введіть ваше ім'я"
        value={name}
        onChangeText={setName}
      />
      <Text>Привіт, {name || 'незнайомець'}!</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 20, gap: 12 },
  input: {
    borderWidth: 1,
    borderColor: '#c7c7cc',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
  },
});
```

### Головна відмінність від вебу

- Подія називається **`onChangeText`**, а не `onChange`. Вона одразу передає готовий рядок (`string`), а не об'єкт події — не треба писати `e.target.value`.
- `value` обов'язково має братися зі стану — якщо забути прив'язати `value`, поле стане **некерованим**, і React Native не зможе синхронізувати те, що бачить користувач, з логікою застосунку.
- Немає CSS-стилізації через клас — рамка, відступи, кольори задаються так само, як у будь-якому іншому компоненті, через `style`.

---

## Типи клавіатур та спеціальні поля

Мобільна клавіатура — це окрема сутність, яку можна підлаштувати під тип даних, що вводяться:

```tsx
<TextInput
  placeholder="Email"
  keyboardType="email-address"
  autoCapitalize="none"
  autoCorrect={false}
  value={email}
  onChangeText={setEmail}
/>

<TextInput
  placeholder="Телефон"
  keyboardType="phone-pad"
  value={phone}
  onChangeText={setPhone}
/>

<TextInput
  placeholder="Пароль"
  secureTextEntry
  value={password}
  onChangeText={setPassword}
/>
```

### Найкорисніші властивості

| Властивість | Призначення |
|---|---|
| `keyboardType` | тип клавіатури: `'default'`, `'numeric'`, `'email-address'`, `'phone-pad'`, `'decimal-pad'` |
| `secureTextEntry` | приховує введений текст (паролі) |
| `autoCapitalize` | `'none'` / `'sentences'` / `'words'` / `'characters'` |
| `autoCorrect` | вмикає/вимикає автовиправлення (зазвичай `false` для email, логінів) |
| `maxLength` | обмеження довжини введення |
| `editable` | `false` — робить поле недоступним для редагування |
| `returnKeyType` | текст на кнопці клавіатури: `'done'`, `'next'`, `'search'` |

---

## Багаторядкове поле та KeyboardAvoidingView

```tsx
<TextInput
  placeholder="Опис"
  multiline
  numberOfLines={4}
  style={{ height: 100, textAlignVertical: 'top' }}
  value={description}
  onChangeText={setDescription}
/>
```

- `multiline` перетворює поле на текстову область (аналог `<textarea>`).
- `textAlignVertical: 'top'` (лише Android) потрібен, щоб текст починався зверху, а не по центру.

### Компонент KeyboardAvoidingView

Часта проблема при роботі з формою у мобільних додатках полягає у тому, що клавіатура на мобільному пристрої **перекриває нижню частину екрана**, і поле вводу, розташоване внизу, стає невидимим. Вирішується обгорткою `KeyboardAvoidingView`:

```tsx
import { KeyboardAvoidingView, Platform } from 'react-native';

<KeyboardAvoidingView
  style={{ flex: 1 }}
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
>
  {/* форма з полями вводу */}
</KeyboardAvoidingView>
```

`behavior` навмисно різний для iOS і Android — ще один приклад того, що платформи поводяться неоднаково і код іноді доводиться розгалужувати через `Platform.OS`.

---

## Проста валідація введення

На старті достатньо звичайних JS-перевірок без бібліотек:

```tsx
import { useState } from 'react';
import { View, TextInput, Text, Pressable, StyleSheet } from 'react-native';

export default function EmailForm() {
  const [email, setEmail] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = () => {
    if (!email.trim()) {
      setError('Поле не може бути порожнім');
      return;
    }
    if (!email.includes('@')) {
      setError('Введіть коректний email');
      return;
    }
    setError('');
    console.log('Форма відправлена:', email);
  };

  return (
    <View style={styles.container}>
      <TextInput
        style={[styles.input, error && styles.inputError]}
        placeholder="Email"
        value={email}
        onChangeText={(text) => {
          setEmail(text);
          if (error) setError(''); // прибираємо помилку одразу при виправленні
        }}
      />
      {error ? <Text style={styles.errorText}>{error}</Text> : null}

      <Pressable style={styles.button} onPress={handleSubmit}>
        <Text style={styles.buttonText}>Надіслати</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 20, gap: 8 },
  input: { borderWidth: 1, borderColor: '#c7c7cc', borderRadius: 8, padding: 12 },
  inputError: { borderColor: '#ff3b30' },
  errorText: { color: '#ff3b30', fontSize: 13 },
  button: { backgroundColor: '#007AFF', padding: 12, borderRadius: 8, alignItems: 'center', marginTop: 8 },
  buttonText: { color: '#fff', fontWeight: '600' },
});
```

Ключова ідея: помилка — це просто ще один рядок у стані (`error`), а її відображення — умовний рендеринг, той самий патерн, що вже відомий студентам.

---

## FlatList: базовий синтаксис

`FlatList` — головний компонент для виведення списків у React Native. На відміну від вебу, де можна просто зробити `.map()` по масиву всередині `<ScrollView>`, для великих або динамічних списків це **не рекомендується**: `FlatList` рендерить лише видимі елементи (віртуалізація), що критично для продуктивності.

```tsx
import { FlatList, View, Text, StyleSheet } from 'react-native';

const fruits = [
  { id: '1', name: 'Яблуко' },
  { id: '2', name: 'Банан' },
  { id: '3', name: 'Вишня' },
];

export default function FruitList() {
  return (
    <FlatList
      data={fruits}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => (
        <View style={styles.row}>
          <Text style={styles.text}>{item.name}</Text>
        </View>
      )}
    />
  );
}

const styles = StyleSheet.create({
  row: { padding: 16, borderBottomWidth: 1, borderBottomColor: '#e5e5ea' },
  text: { fontSize: 16 },
});
```

### Три обов'язкові атрибути

- **`data`** — масив елементів для відображення.
- **`renderItem`** — функція, що повертає JSX для кожного елемента; отримує об'єкт `{ item, index }`.
- **`keyExtractor`** — функція, що повертає унікальний рядковий ключ для кожного елемента (аналог `key` у `.map()` в React).

---

## Корисні властивості FlatList

```tsx
<FlatList
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <ItemCard item={item} />}
  ItemSeparatorComponent={() => <View style={styles.separator} />}
  ListEmptyComponent={<Text style={styles.empty}>Список порожній</Text>}
  ListHeaderComponent={<Text style={styles.header}>Мій список</Text>}
  contentContainerStyle={{ padding: 16 }}
  refreshing={isRefreshing}
  onRefresh={handleRefresh}
/>
```

| Властивість | Призначення |
|---|---|
| `ItemSeparatorComponent` | розділювач між елементами (не рендериться перед першим і після останнього) |
| `ListEmptyComponent` | що показати, якщо `data` — порожній масив (наприклад, "Список порожній") |
| `ListHeaderComponent` / `ListFooterComponent` | елемент(и) до/після списку — заголовок, лічильник, кнопка "завантажити ще" |
| `contentContainerStyle` | стилі для внутрішнього контейнера (відступи всього списку), на відміну від `style`, який стилізує сам скрол-контейнер |
| `refreshing` + `onRefresh` | "потягни, щоб оновити" (pull-to-refresh) — типовий мобільний патерн |
| `horizontal` | горизонтальний список замість вертикального (наприклад, каруселі) |

### Чому не `.map()`

```tsx
// ❌ Небажано для великих/динамічних списків
<ScrollView>
  {items.map((item) => (
    <ItemCard key={item.id} item={item} />
  ))}
</ScrollView>

// ✅ FlatList рендерить лише видимі елементи
<FlatList data={items} keyExtractor={(i) => i.id} renderItem={...} />
```

`.map()` усередині `ScrollView` рендерить **усі** елементи одразу, навіть ті, що поза екраном — на списку з сотнями елементів це помітно "просідає" продуктивність. `FlatList` вирішує це "з коробки".

---

## SectionList: списки з групами

Коли елементи потрібно групувати із заголовками (наприклад, контакти за літерами алфавіту, замовлення за датами) — використовується `SectionList`:

```tsx
import { SectionList, View, Text, StyleSheet } from 'react-native';

const DATA = [
  { title: 'Сьогодні', data: ['Купити хліб', 'Зателефонувати мамі'] },
  { title: 'Учора', data: ['Прибрати кімнату'] },
];

export default function TasksBySection() {
  return (
    <SectionList
      sections={DATA}
      keyExtractor={(item, index) => item + index}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text>{item}</Text>
        </View>
      )}
      renderSectionHeader={({ section: { title } }) => (
        <Text style={styles.sectionHeader}>{title}</Text>
      )}
    />
  );
}

const styles = StyleSheet.create({
  item: { padding: 12, borderBottomWidth: 1, borderBottomColor: '#e5e5ea' },
  sectionHeader: { fontSize: 14, fontWeight: '700', backgroundColor: '#f2f2f7', padding: 8 },
});
```

Різниця з `FlatList` — дані передаються не єдиним масивом (`data`), а масивом **секцій** (`sections`), кожна з яких має `title` і власний `data`.

---

## Комплексний приклад: форма + список

Наступний приклад поєднує все з цього заняття в одному екрані: `TextInput` для введення нового елемента, кнопка додавання, `FlatList` для відображення, а також базова валідація й `KeyboardAvoidingView`.

```tsx
import { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  Pressable,
  FlatList,
  KeyboardAvoidingView,
  Platform,
  StyleSheet,
} from 'react-native';
import { Ionicons } from '@expo/vector-icons';

type Task = { id: string; title: string };

export default function TaskListScreen() {
  const [input, setInput] = useState('');
  const [error, setError] = useState('');
  const [tasks, setTasks] = useState<Task[]>([
    { id: '1', title: 'Переглянути лекцію' },
    { id: '2', title: 'Зробити домашнє завдання' },
  ]);

  const handleAdd = () => {
    if (!input.trim()) {
      setError('Введіть текст завдання');
      return;
    }
    setTasks((prev) => [{ id: Date.now().toString(), title: input.trim() }, ...prev]);
    setInput('');
    setError('');
  };

  const handleDelete = (id: string) => {
    setTasks((prev) => prev.filter((task) => task.id !== id));
  };

  return (
    <KeyboardAvoidingView
      style={styles.screen}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <Text style={styles.title}>Мої завдання</Text>

      <View style={styles.inputRow}>
        <TextInput
          style={[styles.input, error && styles.inputError]}
          placeholder="Нове завдання"
          value={input}
          onChangeText={(text) => {
            setInput(text);
            if (error) setError('');
          }}
          onSubmitEditing={handleAdd}
          returnKeyType="done"
        />
        <Pressable style={styles.addButton} onPress={handleAdd}>
          <Ionicons name="add" size={22} color="#fff" />
        </Pressable>
      </View>
      {error ? <Text style={styles.errorText}>{error}</Text> : null}

      <FlatList
        data={tasks}
        keyExtractor={(item) => item.id}
        contentContainerStyle={{ paddingTop: 12 }}
        ItemSeparatorComponent={() => <View style={styles.separator} />}
        ListEmptyComponent={<Text style={styles.empty}>Завдань поки немає</Text>}
        renderItem={({ item }) => (
          <View style={styles.row}>
            <Text style={styles.rowText}>{item.title}</Text>
            <Pressable onPress={() => handleDelete(item.id)}>
              <Ionicons name="trash-outline" size={20} color="#ff3b30" />
            </Pressable>
          </View>
        )}
      />
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  screen: { flex: 1, padding: 20, backgroundColor: '#fff' },
  title: { fontSize: 22, fontWeight: '700', marginBottom: 16 },
  inputRow: { flexDirection: 'row', gap: 8 },
  input: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#c7c7cc',
    borderRadius: 8,
    padding: 12,
  },
  inputError: { borderColor: '#ff3b30' },
  errorText: { color: '#ff3b30', fontSize: 13, marginTop: 4 },
  addButton: {
    backgroundColor: '#007AFF',
    width: 44,
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 12,
  },
  rowText: { fontSize: 15 },
  separator: { height: 1, backgroundColor: '#e5e5ea' },
  empty: { textAlign: 'center', color: '#8e8e93', marginTop: 40 },
});
```
