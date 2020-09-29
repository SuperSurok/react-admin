# React-admin

 [Data Provider](#Data-Provider)
   * [Описание](#Описание)
   * [Использование](#Использование)
  
## Описание (Проводник данных)
  
Каждый раз, когда react-admin необходимо взаимодействовать с API он вызвает методы объекта dataProvider:
```js
dataProvider
  .getOne('posts', { id: 123 })
  .then(response => {
      console.log(response.data); // { id: 123, title: "hello, world" }
});
```
Задача dataProvider заключается в том, чтобы обернуть эти вызовы метода в HTTP запрос и трансформировать HTTP ответ в формат данных ожидаемых reac-admin.
Технически dataProvider это адаптер для API.

Включить проводник можно, передав его как свойство в компонент ```<Admin>```:
```js
  import { Admin, Resource } from 'react-admin';
  import dataProvider from '../myDataProvider';

  const App = () => (
    <Admin dataProvider={dataProvider}>
        // ...
    </Admin>
  )
```
Этот адаптер может взаимодействовать с любым API, неважно что используется REST, GraphQL, RPC и т.д.
Проводник так же идеальное место для добавления пользовательских HTTP заголовков, аутенитификации и т.д.

Проводник обязательно должен иметь следующие методы:
```js
  const dataProvider = {
    getList:    (resource, params) => Promise,
    getOne:     (resource, params) => Promise,
    getMany:    (resource, params) => Promise,
    getManyReference: (resource, params) => Promise,
    create:     (resource, params) => Promise,
    update:     (resource, params) => Promise,
    updateMany: (resource, params) => Promise,
    delete:     (resource, params) => Promise,
    deleteMany: (resource, params) => Promise,
  }
```
Описание методов dataProvider:
  <table>
      <tr><td>getList</td><td>Поиск ресурса</td><td>{ pagination: { page: {int} , perPage: {int} }, sort: { field: {string}, order: {string} }, filter: {Object} }</td></tr>
      <tr><td>getOne</td><td>Взять один ресурс по id</td><td>{ id: {mixed} }</td></tr>
      <tr><td>getMany</td><td>Взять список ресурсов по их id</td><td>{ ids: {mixed[]} }</td></tr>
      <tr><td>getManyReference</td><td>Получить список ресурсов относящихся к другому ресурсу</td><td>{ target: {string}, id: {mixed}, pagination: { page: {int} , perPage: {int} }, sort: { field: {string}, order: {string} }, filter: {Object} }</td></tr>
      <tr><td>create</td><td>Создать один ресурс</td><td>{ data: {Object} }</td></tr>
      <tr><td>update</td><td>Обновить один ресурс</td><td>{ id: {mixed}, data: {Object}, previousData: {Object} }</td></tr>
      <tr><td>updateMany</td><td>Обновить множество ресурсов</td><td>{ ids: {mixed[]}, data: {Object} }</td></tr>
      <tr><td>delete</td><td>Удалить один ресурс</td><td>{ id: {mixed}, previousData: {Object} }</td></tr>
      <tr><td>deleteMany</td><td>Удалить множество ресурсов</td><td>{ ids: {mixed[]} }</td></tr>
</table>

Примеры:
```js
dataProvider.getList('posts', {
  pagination: { page: 1, perPage: 5 },
  sort: { field: 'title', order: 'ASC' },
  filter: { author_id: 12 },
});
dataProvider.getOne('posts', { id: 123 });
dataProvider.getMany('posts', { ids: [123, 124, 125] });
dataProvider.getManyReference('comments', {
    target: 'post_id',
    id: 123,
    sort: { field: 'created_at', order: 'DESC' }
});
dataProvider.update('posts', {
    id: 123,
    data: { title: "hello, world!" },
    previousData: { title: "previous title" }
});
dataProvider.updateMany('posts', {
    ids: [123, 234],
    data: { views: 0 },
});
dataProvider.create('posts', { data: { title: "hello, world" } });
dataProvider.delete('posts', {
    id: 123,
    previousData: { title: "hello, world" }
});
dataProvider.deleteMany('posts', { ids: [123, 234] });
```

  
