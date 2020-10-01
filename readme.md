# React-admin

1. [Data Provider](#Data-Provider)
    * [Описание](#Описание)
    * [Формат запроса](#request-format)

1. [Admin](#admin)
1. [Resource](#resource)
1. [List View](#list-view)
1. [List Component](#list-component)  
1. [ListGuesser](#ListGuesser)
1. [useListContext](#uselistcontext)
1. [ListBase](#ListBase)
1. [useListController](#useListController)

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
## List View

```<List>``` отображает список записей полученных из API. Когда данные попадают в ```ListContext``` они становяться доступными для потомков, обычно ```<Datagrid>```, который затем делегирует рендеринг каждого свойства записи компоненту ```<Field>```.

## List Component
```<List>``` принимает список записей из ```dataProvider``` и отображает макет (заголовок, кнопки, фильтры, пагинацию). Он делегирует рендеринг списка записей своим дочерним компонентам. Обычно за это отвечает ```<Datagrid>```, отображает таблицу с одной строкой для каждой записи.

Список свойств компонента ```<List>```:
* title
* actions
* exporter
* bulkActions
* filters - отображает фильтр компонента
* filterDefaultValues - значение по умолчанию для фильтра
* perPage
* sort
* filter
* pagination
* aside
* empty

Minimal code to display list:
```js
 import * as React from "react";
 import { Admin, Resource } from 'react-admin';
 import jsonServerProvider from 'ra-data-json-server';

 import { PostList } from './posts';

 const App = () => (
     <Admin dataProvider={jsonServerProvider('https://jsonplaceholder.typicode.com')}>
         <Resource name="posts" list={PostList} />
     </Admin>
 );

 export default App;

 // in src/posts.js
 import * as React from "react";
 import { List, Datagrid, TextField } from 'react-admin';

 export const PostList = (props) => (
     <List {...props}>
         <Datagrid>
             <TextField source="id" />
             <TextField source="title" />
             <TextField source="body" />
         </Datagrid>
     </List>
 );
```

## ListGuesser

Вместо использования настраиваемого ```<List>``` можно использовать ```<ListGuesser>```, который сам определяет на основе полученных данниых из API, какие поля нужно отобразить. Так же он отображает в консоли компонент  ```<Datagrid>``` с дочерними компонентами. Можно скопировать необходимые поля в свой код.

## useListContext

Компонент ```<List>``` заботится о получении данных и передаёт эти данные в контекст называемый ListContext, что делает их доступными для потомков. По сути он передаёт большое количество переменных в контекст. Любой компонент может получить данные, используя ```useContextHook``` hook. По-факту ```<Datagrid>```, ```<Filter>```, ```<Pagination>``` используют ```useContextHook``` hook.

## ListBase

В добавок к получению данных компонент ```<List>``` отображает заголовок страницы, действия, контент и боковое меню. ```<ListBase>``` позволяет создать свой собственный ```<List>```.

## useListController

Как объяснялось ранее, ```<ListBase>``` получает данные и передаёт их в ```<ListContext>``` и затем отображает своих детей. По-факту код ```<ListBase>``` очень простой: 
```js
 import * as React from 'react';
 import { useListController, ListContextProvider } from 'react-admin';

 const ListBase = ({ children, ...props }) => (
     <ListContextProvider value={useListController(props)}>
         {children}
     </ListContextProvider>
 );

export default ListBase;
```
Как видно, часть контроллера представления списка обрабатывается хуком ```]useListController```. Если не хочется ипользовать ```<ListContext>``` в кастомном списке, можно использовать ```useListContoller```, чтоба напрямую полчить доступ к аднным списка. Он возваращает такой же объект как и ```useListContext```.

Если кастомный компонент не использует ```ListContextProvider```, нельзя использовать ```<Datagrid>```, ```<Simplelist>```, ```<Pagination>``` и т.д. Все эти компоненты полагаются на ```ListContext```.
