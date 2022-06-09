# Блокчейн хранилище

Что это? Это смарт-контракт созданный для хранения данных. Для более удобной работы с данными и удешевления использования они хранятся в виде обычных строк, которые нужно изменять за пределами контракта. Все что он делает это сохраняет, меняет, возвращает и удаляет строк(у|и). А за то что в них находится отвечает фронтенд сторона. На данный момент, для привязки данных приложения к самому приложению, мы используем название домена.

- [Код контракта](https://github.com/NotEternal/contracts/blob/main/contracts/Storage.sol)
- Главный задеплоиный контракт для использования находится на **Binance Smart Chain**: [Storage](https://bscscan.com/address/0xa7472f384339d37efe505a1a71619212495a973a#code=)

Можно работать с любыми библиотеками на фронте, но при этом нужно использовать главный контракт хранилища. Этим мы достигнем простоты изменения, переноса и контроля над данными. Будет проблемой, если использовать много контрактов на разных сетях или менять что-то в нем только для себя. Главное думать о совместимости. Нужно что-то изменить в контракте? Тогда делаем новый для всего что его использует.

## Сохранение и загрузка данных в приложении

Общая схема работы:

```
интерфейс приложения <-> внутренний код <-> хранилище
```

Если мы хотим видеть сразу увидеть изменения в интерфейсе, после сохранения данных, то нужно в ручную следить за изменением данных в контракте. Пример этой схемы разберем на логотипе: где-то в приложении есть логотип и нужно менять его в зависимости от того что выбрал пользователь.

> Приблизительный код с примером на [React](https://reactjs.org/). Много частей уменьшенно и упрощенно.

1. В интерфейсе добавляем `input` для адреса логотипа
2. Добавляем кнопку, которая будет сохранять адрес

```jsx
<input value={logoUrl} onChange={onLogoUrl} />
<button onClick={saveLogo}>Save</button>
```

3. Добавляем обработчик сохранения:

```js
const saveLogo = async () => {
  await saveData({
    provider: library.provider,
    owner: account,
    data: {
      logo: logoUrl,
    },
  });
};
```

4. Где-то создаем функцию `saveData` (можно делать все из компонентов, но лучше разделить функционал и делать это в других частях кода):

```js
import Web3 from "web3";
import { STORAGE_ADDRESS } from "./constants";
import { Storage } from "./abi";

export const saveData = async ({ provider, owner, data }) => {
  const web3 = new Web3(provider);
  const storage = new web3.eth.Contract(Storage.abi, STORAGE_ADDRESS);
  const currentDomain = window.location.hostname;

  return storage.methods.setKeyData(currentDomain, {
    owner,
    info: JSON.stringify(data),
  });
};
```

5. Начинаем загрузку данных из хранилища. Cоздаем основную функцию:

```js
export const fetchData = async (provider) => {
  const web3 = new Web3(provider);
  const storage = new web3.eth.Contract(Storage.abi, STORAGE_ADDRESS);
  const currentDomain = window.location.hostname;

  const { info, owner } = await storage.methods.getData(currentDomain).call();

  return { ...JSON.parse(info), admin: owner };
};
```

6. Теперь нужно сохранить запрошенные данные. Используем [Redux для React](https://react-redux.js.org/) + [Redux toolkit](https://redux-toolkit.js.org/) (изменяем место хранения и использование в зависимости от структуры приложения):

```js
// К примеру этот запрос будет в App.tsx
useEffect(() => {
  const fetch = async () => {
    const data = await fetchData(library.provider);

    dispatch(retrieveData(data));
  };

  fetch();
}, [chainId, library]);

...

// в основном файле mainReducer.ts
export default createReducer(initialState, (builder) =>
  builder.addCase(retrieveData, (state, action) => {
    const data = action.payload;

    state.logo = data.logo
  })
);
```

7. Можем использовать логотип в интерфейсе:

```jsx
import { useSelector } from "react-redux";

function Header() {
  const { logo } = useSelector((state) => state);

  return (
    <div>
      <img src={logo} />
      ...
    </div>
  );
}
```

## Ответы на популярные вопросы

### Разница между обычной базой данных и контрактом?

База данных это структура, которая может быть заранее готова (со стороны базы). Там есть заранее готовый интерфейс для работы с данными. С другой стороны контракт для хранения не обрабатывает данные или их части, так-как данные представлены строкой. Он может только добавлять или удалять строки целиком.

- Почему это лучше? Потому что ...?
- Почему обязательно использовать блокчейн? Во первых потому что мы работаем со смарт-контрактом, что означает разработку в блокчейне. Во вторых в связи со устройством блокчейна мы не храним данные в одной точке, что увеличивает безопасность.
- Почему именно этот контракт? Потому что это лучшее что есть на данный момент. Думаю не будет проблемой начать использовать другое решение, если оно появится.

### Как хранятся данные?

Весь код отвечающий за хранение:

```js
struct Data {
    address owner;
    string info;
}

mapping(string => Data) private allData;
```

Детали:

- `struct Data`: тип описывающий данные. Включает в себя `owner` ключ содержащий адрес акаунта (может изменять данные) и `info` ключ содержащий строку с данными
- `string => ...`: ключ в виде строки, по которому получаем доступ к данным
- `... => Data`: значение в виде данных

### На сколько безопасно хранятся данные?

Все работает в блокчейне, что означает децентрализованную и криптографически защищенную систему. Данные хранятся распределенно на множестве компьютеров и их возможно изменить только через указанные правила. Их изменение возможно через указанный адрес, поэтому подделать их невозможно.

### Как использовать контракт напрямую?

Два основных метода для работы c хранилищем:

1. Сохранение данных:

```js
function setKeyData(string memory _key, Data memory _data) external;
```

Первым параметром передаем ключ в виде строки (домен приложения). Вторым обьект в виде `Data` структуры:

```js
{
    address owner;
    string info;
}
```

2. Получение данных:

```js
function getData(string memory _key) external view returns(Data memory);
```

Передаем параметр в виде строки (домен) и получаем обьект с адресом владельца и данными.

3. Удаление:

```js
function clearKeyData(string memory _key) external;
```

Дополнительные методы:

```js
// получаем все сохраненные ключи для данных
function allKeys() external view returns(string[] memory);
// получаем все сохраненные данные
function allKeysData() external view returns(Data[] memory);
// можно сохранить сразу несколько обьектов с данными
function setKeysData(KeyData[] memory _keysData) external;
// удаляем множество обьектов
function clearKeysData(string[] memory _keys) external;
```

### Как использовать контракт в приложении?

Мы можем использовать одну из библиотек для взаимодействия с EVM блокчейном:

- [Web3.js](https://web3js.readthedocs.io/)
- [Ethers](https://docs.ethers.io/)

Для использования хранилища нужно иметь:

- Одна из библиотек выше
- Адрес контракта
- ABI ([Application Binary Interface](https://docs.soliditylang.org/en/develop/abi-spec.html)): JSON интерфейс с помощью которого мы создаем обьект описывающий смарт-контракт (свойства, методы и тд.)

1. Получаем адрес контракта в блокчейне. Используем контракт с BSС: [Хранилище](https://bscscan.com/address/0xa7472f384339d37efe505a1a71619212495a973a)
2. Получаем ABI: [Ccылка](https://raw.githubusercontent.com/noxonsu/unifactory/main/src/contracts/build/Storage.json)
3. Делаем новый экземпляр контракта из полученных данных:

```js
import web3 from "web3";
import { Storage } from "./abis";
import { STORAGE_ADDRESS } from "./constants";
// если мы хотим изменить данные, тогда нужно передать провайдера
// из внешнего кошелька
const web3 = new Web3("https://bsc-dataseed.binance.org");
const storage = new web3.eth.Contract(Storage.abi, STORAGE_ADDRESS);
```

4. Можно использовать контракт:
   > если работаем с внешним кошельком, нужно находиться на той же сети с которой был создан экземпляр контракта. В данном примере это BSC

```js
storage.methods.setKeyData("example.com", {
  owner: "0x...",
  info: JSON.stringify({
    foo: 1,
    bar: 2,
    subObj: {
      foo: 3,
    },
  }),
});

storage.methods.getData("example.com");
// ...
```

Если публичного доступа к ABI нет, но есть файлы контракта, тогда нужно сделать его самостоятельно. Для этого можно использовать:

- [Remix IDE](https://remix.ethereum.org/): все делается в браузере. Подходит для небольших проектов из 1 или нескольких контрактов
- [Truffle](https://trufflesuite.com/): набор инструментов для разработки, деплоя и тестирования контрактов
- [Hardhat](https://hardhat.org/): более продвинутый инструмент

TODO: дополнить

### Как работать с данными в приложении?

понять как связать это c интерфейсом, какой js сторадж использовать (я хз что это, но мы помню долго спорили) - реакт-контекст или мобикс или что

### Нужно ли делать панель админа в том же дизайне как и главное приложение?

надо ли интегрировать дизайн админки внутрь дапки чтоб был в том же стиле

### Можно ли сделать одну админ панель и использовать во всех приложениях?

почему бы не сделать сделать одну универсальную админку типа как [bootstrap](https://themes.getbootstrap.com/product/hyper-responsive-admin-dashboard-template/)
