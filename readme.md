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
1. [Datagrid](#datagrig)
      * [body](#body)
      * [rowStyle](#rowstyle)
      * [rowClick](#rowClick)
      * [expand](#expand)
      * [isRowSelectable](#isrowslectable)
      * [CSS](#css)
1. [SimpleList](#simplelist)

## Описание (Проводник данных)
  
Каждый раз, когда react-admin необходимо взаимодействовать с API он вызвает методы объекта dataProvider:
```js
dataProvider
  .getOne('posts', { id: 123 })
  .then(response => {
      console.log(response.data); // { id: 123, title: "hello, world" }
});
```
Задача dataProvider заключается в том, чтобы обернуть эти вызовы метода в HTTP запрос и трансформировать HTTP ответ в формат данных ожидаемых react-admin.
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
Пример реального кода:
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

Компонент ```<Admin>```создаёт приложение со своим собственным состоянием, роутингом и контроллером логики.
Минимальный набор свойств для ```<Admin>``` свойство ```dataProvider``` и как минимум один дочерний ```<Resource>```.

Пример:
```js
import * as React from "react";

import { Admin, Resource } from 'react-admin';
import simpleRestProvider from 'ra-data-simple-rest';

import { PostList } from './posts';

const App = () => (
    <Admin dataProvider={simpleRestProvider('http://path.to.my.api')}>
        <Resource name="posts" list={PostList} />
    </Admin>
);
```

## Resource

Компонент ```<Resource>``` отображает данные полученные из API и предоставляет возможность работы с CRUD (create-read-update-delete).

Свойства для работы с интрефейсом CRUD:
 * list - если определён, название ресурса отображается в меню
 * create - созданиие новой сущности
 * edit - возможность редактирования сущности
 * show - показать отдельную сущность
 
Пример:
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

```<List>``` отображает список записей полученных из API. Когда данные попадают в ```ListContext``` они становятся доступными для потомков, обычно ```<Datagrid>```, который затем делегирует рендеринг каждого свойства записи компоненту ```<Field>```.

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

Минимальный код для отображения списка:
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

#### Page Title
По умолчанию заголовок для списка это "[resource] list" => "Posts list". Чтобы изменить его нужно использовать свойство ```title```:
```js
export const PostList = (props) => (
    <List {...props} title="List of posts">
        ...
    </List>
);
```
#### Actions
Можно заменить дефолтные действия со списком на свои, используя свойство ```actions```.

Пример:
```js
import * as React from 'react';
import { cloneElement, useMemo } from 'react';
import PropTypes from 'prop-types';
import {
    useListContext,
    TopToolbar,
    CreateButton,
    ExportButton,
    Button,
    sanitizeListRestProps,
} from 'react-admin';
import IconEvent from '@material-ui/icons/Event';

const ListActions = (props) => {
    const {
        className,
        exporter,
        filters,
        maxResults,
        ...rest
    } = props;
    const {
        currentSort,
        resource,
        displayedFilters,
        filterValues,
        hasCreate,
        basePath,
        selectedIds,
        showFilter,
        total,
    } = useListContext();
    return (
        <TopToolbar className={className} {...sanitizeListRestProps(rest)}>
            {filters && cloneElement(filters, {
                resource,
                showFilter,
                displayedFilters,
                filterValues,
                context: 'button',
            })}
            <CreateButton basePath={basePath} />
            <ExportButton
                disabled={total === 0}
                resource={resource}
                sort={currentSort}
                filterValues={filterValues}
                maxResults={maxResults}
            />
            {/* Add your custom actions */}
            <Button
                onClick={() => { alert('Your custom action'); }}
                label="Show calendar"
            >
                <IconEvent />
            </Button>
        </TopToolbar>
    );
};

export const PostList = (props) => (
    <List {...props} actions={<ListActions />}>
        ...
    </List>
);
```

Используя кастомный ```<ListActions>``` можно отключить или переопределить порядок кнопок на основании прав доступа. Для этого нужно передать ```permissions``` вниз из компонента ```<List>```:
```js
export const PostList = ({ permissions, ...props }) => (
    <List {...props} actions={<PostActions permissions={permissions} {...props} />}>
        ...
    </List>
);
```
#### Exporter
Существует дефолтная возможность экспорта файлов - кнопка ```<ExportButton>```. Кнопка неактивна, если в текущем ```<List>``` нет данных.

Действие конопки по умолчанию:
   1. Вызов dataProvider с сортировкой и фильтром(без пагинации)
   1. Трансформация в SVG
   1. Загрузка CSV файла

### ListGuesser

Вместо использования настраиваемого ```<List>``` можно использовать ```<ListGuesser>```, который сам определяет на основе полученных данных из API, какие поля нужно отобразить. Так же он отображает в консоли компонент  ```<Datagrid>``` с дочерними компонентами. Можно скопировать необходимые поля в свой код.

### useListContext

Компонент ```<List>``` заботится о получении данных и передаёт эти данные в контекст называемый ListContext, что делает их доступными для потомков. По сути он передаёт большое количество переменных в контекст. Любой компонент может получить данные, используя ```useContextHook``` hook. По-факту ```<Datagrid>```, ```<Filter>```, ```<Pagination>``` используют ```useContextHook``` hook.

### ListBase

В добавок к получению данных компонент ```<List>``` отображает заголовок страницы, действия, контент и боковое меню. ```<ListBase>``` позволяет создать свой собственный ```<List>```.

### useListController

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
Как видно, часть контроллера представления списка обрабатывается хуком ```useListController```. Если не хочется использовать ```<ListContext>``` в кастомном списке, можно использовать ```useListContoller```, чтобы напрямую получить доступ к данным списка. Он возваращает такой же объект как и ```useListContext```.

Если кастомный компонент не использует ```ListContextProvider```, нельзя использовать ```<Datagrid>```, ```<Simplelist>```, ```<Pagination>``` и т.д. Все эти компоненты полагаются на ```ListContext```.

## Datagrig

Компонент Datagrid отображает список записей в виде таблицы. Обычно он используется как потомок компонентов ```<List>``` и ```<ReferenceManyField>```. Без этих компонентов он должен использоваться внутри ```<ListContext>```.

Свойства компонента ```<Datagrid>```:
   * ```body```
   * ```rowStyle```
   * ```rowClick```
   * ```expand```
   * ```isRowSelectable```
   
Он отображается, как множество колонок и принимает ```<Field>``` в качестве детей. Использует поле ```<label>``` для заголовка колонки. Для полей без ```label``` в качестве заголовка используется ```source```.

Пример:
```js
   import * as React from "react";
   import { List, Datagrid, TextField, EditButton } from 'react-admin';

   export const PostList = (props) => (
      <List {...props}>
         <Datagrid>
            <TextField source="id" />
            <TextField source="title" />
            <TextField source="body" />
            <EditButton />
           </Datagrid>
      </List>
   );
```

```<Datagrid>``` - это компонент итератор. Он получает массив id и данные из ```ListContex``` и проходит по id отображая каждую запись. Другим примером компонента итератора является ```<SingleFieldList>```.

### body

По умолчанию ```<Datagrid>``` отображет тело(данные), используя служебный компонент ```<DatagridBody>```. Можно передать свой собственнй компонент в ```body```.
У ```<DatagridBody>``` есть свойство ```row``` настроенное через ```<DatagridRow>``` по умолчанию для тех же целей. ```<DatagridRow>``` принимает запись ```record``` для строки, ```resource``` и копию детей ```Datagrid```. Это позволяет создавать собственную логику без копирования нескольких

### rowStyle

Можно здать собсвенные стили для для строки ```<Datagrid>```, применяется к ```<tr>```. ```rowStyle``` ожидает функцию, которая применяется для каждой строки, передавая текущую запись и индекс как аргументы. Функция возврщает объект со стилями.

Пример:
```js
   const postRowStyle = (record, index) => ({
      backgroundColor: record.nb_views >= 500 ? '#efe' : 'white',
   });
   export const PostList = (props) => (
      <List {...props}>
         <Datagrid rowStyle={postRowStyle}>
            ...
        </Datagrid>
    </List>
);
```

### rowClick

Можно ловить клики по строкам и перенаправлять их, чтобы показать внутреннее отображение строки или для редактирования.
```rowClick``` принимает следующие значения:
   * "edit" - направит на редактирование
   * "show" - покажет новое окно
   * "expand" - откроет ```expand``` панель
   * "toggleSelection" - для запуска ```onToggleItem```
   * function(id, basePath, record) => path для редиректа по кастомному пути
   
### expand

Чтобы показать больше данных, не добавляя множество колонок, можно показать их в панели расширения, расширяя строку вниз по клику. Для этого используется свойство ```expand```. 

Пример:
```js

const PostPanel = ({ id, record, resource }) => (
    <div dangerouslySetInnerHTML={{ __html: record.body }} />
);

const PostList = props => (
    <List {...props}>
        <Datagrid expand={<PostPanel />}>
            <TextField source="id" />
            <TextField source="title" />
            <DateField source="published_at" />
            <BooleanField source="commentable" />
            <EditButton />
        </Datagrid>
    </List>
)
```

### isRowSelectable

С помощью этого свойства для строки можно добавить чекбокс. Оно ожидает функцию, которая будет передавать запись для каждого ```<DatagridRow>``` и возращать булево выражение.
В этом примере чекбоксы будут показаны для колонок шире 300:
```js
export const PostList = props => (
    <List {...props}>
        <Datagrid isRowSelectable={ record => record.id > 300 }>
            ...
        </Datagrid>
    </List>
);
```

### CSS
Можно использовать className.
Можно стилизовать используя свойства:
   * ```table```: альтернатива ```className```. Применяется к корневому элементу.
   * ```tbody```: применятеся к tbody.
   * ```headerCell```: применяется к заголовкам ячеек.
   * ```row```: применяется к каждой колонке.
   * ```rowEven```: применятся к четным строкам.
   * ```rowOdd```: применятся к нечётным строкам.
   * ```rowCell``` применяется к кажддой ячейке строки.

Пример:
```js
import * as React from "react";
import { makeStyles } from '@material-ui/core';

const useStyles = makeStyles({
    row: {
        backgroundColor: '#ccc',
    },
});

const PostList = props => {
    const classes = useStyles();
    return (
        <List {...props}>
            <Datagrid classes={{ row: classes.row }}>
                ...
            </Datagrid>
        </List>
    );
}
```

Если нужно переписать стили для хедера и ячейки независимо для каждой колонки, нужно использовать ```headerClassName``` и ```cellClassName``` в ```<Field>```.
Нельзя использовать для обёрнутых компонентов.
Пример:
```js
import * as React from "react";
import { makeStyles } from '@material-ui/core';

const useStyles = makeStyles(theme => ({
    hiddenOnSmallScreens: {
        [theme.breakpoints.down('md')]: {
            display: 'none',
        },
    },
}));

const PostList = props => {
    const classes = useStyles();
    return (
        <List {...props}>
            <Datagrid>
                <TextField source="id" />
                <TextField source="title" />
                <TextField
                    source="views"
                    headerClassName={classes.hiddenOnSmallScreens}
                    cellClassName={classes.hiddenOnSmallScreens}
                />
            </Datagrid>
        </List>
    );
};
```

## SimpleList

Для мобилок ```<Datagrid>``` часто не подходит, не хватает места для отображения нескольких колонок. В этом случае можно использовать простой список с одной колонкой на строке ```<SimpleList>``` и ```<ListItem>```, использующий [material-ui's](#https://material-ui.com/components/lists/). ```<SimpleList>``` - это компонент итератор: получает массив ids и данные из ```<ListContext>``` и, проходя по id отображает каждую запись.

### Свойства
| Свойство | Обязательность | Тип | По-умолчанию | Описание 
| ----- | ----- | ----- | ----- | ----- 
| ```primatyText``` | Обязательно | Функция | - | Передаётся, как свойство ```<ListItemText primary>``` |
| ```secondaryText``` | Опционально | Функция | - | Передаётся, как свойство ```<ListItemText secondary>``` |
| ```tertiaryText``` | Опционально | Функция | - | Передаётся, как комплемент в ```<ListItemText primary>``` с пользовательским стилем |
| ```linkType``` | Опционально |```строка``` | ```Функция``` | ```false``` | редактируется | Цель ссылки ```<ListItem>```. Установить ```false```, чтобы отключить ссылку. При передаче функции ```(record, id) => string``` будет применена ссылка для каждой отдельной записи. |
| ```leftAvatar``` | Опционально | Функция | - | Если установлено, то ```<ListItem>``` отображает ```<ListItemAvatar>``` перед ```<ListItemText>``` |
| ```leftIcon``` | Опционально | Функция | - | Если установлено, то  ```<ListItem>``` отображает ```<ListItemAvatar>``` перед ```<ListItemText>``` |
| ```rightAvatar``` | Опционально | Функция | - | Если установлено, то  ```<ListItem>``` отображает ```<ListItemAvatar>``` после ```<ListItemText>``` |
| ```rightIcon``` | Опционально | Функция | - | Если установлено, то  ```<ListItem>``` отображает ```<ListItemAvatar>``` после ```<ListItemText>``` |
| ```className``` | Опционально | строка | - | Применяется к корневому элементу |

#### Использование
Можно использовать ```<SimpleList>```, как дочерний компонент ```<List>```  или ```<ReferenceManyChild>```:
```js
import * as React from "react";
import { List, SimpleList } from 'react-admin';

export const PostList = (props) => (
    <List {...props}>
        <SimpleList
            primaryText={record => record.title}
            secondaryText={record => `${record.views} views`}
            tertiaryText={record => new Date(record.published_at).toLocaleDateString()}
            linkType={record => record.canEdit ? "edit" : "show"}
        />
    </List>
);
```
