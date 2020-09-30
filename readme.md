# React-admin

 [Data Provider](#Data-Provider)
   * [Описание](#Описание)
   * [Формат запроса](#request-format)
 [Admin](#admin)
 [Resource](#resource)
  
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
### Формат запроса
Стандартные методы
| Метод | Описание | Формат праметров
| ------ | ------ | ------ |
| getList | Поиск ресурса | ```{ pagination: { page: {int} , perPage: {int} }, sort: { field: {string}, order: {string} }, filter: {Object} }``` |
| getOne | Взять один ресурс по id | ```{ id: {mixed} }``` |
| getMany | Взять список ресурсов по их id |  ```ids: {mixed[]} }``` |
| getManyReference | Получить список ресурсов относящихся к другому ресурсу | ```{ target: {string}, id: {mixed}, pagination: { page: {int} , perPage: {int} }, sort: { field: {string}, order: {string} }, filter: {Object} }``` |
| create | Создать один ресурс | ```{ data: {Object} }``` |
| update | Обновить один ресурс | ```{ id: {mixed}, data: {Object}, previousData: {Object} }``` |
| updateMany | Обновить множество ресурсов | ```{ ids: {mixed[]}, data: {Object} }``` |
| delete | Удалить один ресурс | ```{ id: {mixed}, previousData: {Object} }``` |
| deleteMany | Удалить множество ресурсов | ```{ ids: {mixed[]} }``` |

Примеры
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
Пример рабочего кода
```js
 export const dataProvider: DataProvider = {
  getList: (resource: string, params: GetListParams) => {
    switch (resource as KnownResources) {
      case 'entities': {
        const { sort: paramsSort, pagination: paramsPagination } = params;

        const sort = entitySortEncode(paramsSort as SortType);
        const paging = entityPaginationEncode(paramsPagination as PaginationType);
        const filter: EntityDetailsFilter = {
          ...params.filter,
          sort,
          paging,
        };

        return http.post<EntityDetailsView[]>('/entity/admin', filter).then(({ data }) => {
          // NOTE: Do data normalization, because the number of returned elements
          // is (paging.limit + 1) - for 'has more' indication.
          let normalizedData = data;
          const itemPerPage = filter.paging?.limit;
          if (itemPerPage && data.length > itemPerPage) {
            normalizedData = dropRight(data, data.length - itemPerPage);
          }

          return Promise.resolve({ data: normalizedData, total: calcTotal(data.length, paging) });
        });
      }

      case 'accounts':
        return http.get<AccountView[]>(`/usermanager/account/extended`).then(({ data }) => {
          return Promise.resolve({ data, total: data.length });
        });

      case 'assistants':
        return http
          .get<AccountShortView[]>(`/usermanager/account/${params.filter?.id}/assistant/available`)
          .then(({ data }) => {
            return Promise.resolve({ data, total: data.length });
          });

      case 'companies':
        return http.get<CompanyShortView[]>(`/usermanager/company`).then(({ data }) => {
          return Promise.resolve({ data, total: data.length });
        });

      case 'parents':
        return http.get<CompanyShortView[]>(`/usermanager/company`).then(({ data }) => {
          return Promise.resolve({ data: data.filter((item) => item.id !== +params.filter.id), total: data.length });
        });

      default:
        return Promise.reject(`Unknown resource '${resource}'`);
    }
  }
 }
```

## Admin

Компонент ```<Admin>```создаётт приложение со своим собственным состоянием, роутингом и контроллером логики.
```<Admin>``` требуется только свойство ```dataProvider``` и как минимум один дочерний ```<Resource>``` для начала работы.

## Resource

Компонент ```<Resource>``` отображает данные полученные из API и предоставляет возможнось работы с CRUD (create-read-update-delete).
Свойства для работы с интрефейсом CRUD
 * list - если определён, название ресурса отображается в меню
 * create - созданиие новой сущности
 * edit - возможность редактирования сущности
 * show - показать отдельную сущность
 
Пример
```js
import * as React from "react";
import { Admin, Resource } from 'react-admin';
import jsonServerProvider from 'ra-data-json-server';

import { PostList, PostCreate, PostEdit, PostShow, PostIcon } from './posts';
import { UserList } from './posts';
import { CommentList, CommentEdit, CommentCreate, CommentIcon } from './comments';

const App = () => (
    <Admin dataProvider={jsonServerProvider('https://jsonplaceholder.typicode.com')}>
        <Resource name="posts" list={PostList} create={PostCreate} edit={PostEdit} show={PostShow} icon={PostIcon} />
        <Resource name="users" list={UserList} />
        <Resource name="comments" list={CommentList} create={CommentCreate} edit={CommentEdit} icon={CommentIcon} />
        <Resource name="tags" />
    </Admin>
);
```
