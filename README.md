# valtio-persist

`npm i valtio-persist` allows flexible and performant saving of state to disk.

## Quick Start - Basic Usage

```typescript
import proxyWithPersist, { PersistStrategy } from 'valtio-persist';
import { subscribeKey } from 'valtio/utils';

const appStateProxy = proxyWithPersist({
  // must be unique, files/paths will be created with this prefix
  name: 'appState',

  initialState: {
    counter: 0,
  },
  persistStrategies: PersistStrategy.SingleFile,
  version: 0,
  migrations: {},

  // see "Storage Engine" section below
  getStorage: () => storage,
});

console.log('counter:', appStateProxy.counter);

subscribeKey(appStateProxy._persist, 'loaded', (loaded) => {
  if (loaded) {
    console.log('it is now safe to make changes to appStateProxy. the changes will now be persisted.');
  }
});
```

This will persist the entire object into one file, on every change.

You can read from `appStateProxy` immediately, however if you want changes persisted, wait until `appStateProxy._persist.loaded` goes to `true`.

This is obvious but to be safe, keep in mind the base value (`initialState`) must be an object. This applies to `proxy` as well from valtio, the argument to `proxy` is an object.

Every object returned by `proxyWithPersist` gets a special `_persist` key added to it. This key has the value of:

```typescript
{
  status: 'loading' | 'loaded' | 'error';
  loading: boolean;
  loaded: boolean;
  error: null | Error;
}
```

You can use this section to figure out when loading has completed.

## Storage Engine

You can use any storage engine as long as it respects the following interface:

```typescript
export type ProxyPersistStorageEngine = {
  // returns null if file not exists
  getItem: (name: string) => string | null | Promise<string | null>;

  setItem: (name: string, value: string) => void | Promise<void>;
  removeItem: (name: string) => void | Promise<void>;
  getAllKeys: () => string[] | Promise<string[]>;
};
```

`getItem` should return `null` if file or path does not exist.

`getAllKeys` is used for the `PersistStrategy.MultiFile`. If you do not use this strategy, then you can make this function no-op.

To use this engine, set the `getStorage` option to a function that returns this. It can be async, it is only run once.

```typescript

const stateProxy = proxyWithPersist({
  // ...
  getStorage: async () => {

    // do some async stuff, maybe create a directory you want to store this into

    // return storage interface
    return {
      getItem: () => { ... },
      setItem: () => { ... },
      removeItem: () => { ... },
      getAllKeys: () => { ... }
    }
  }
})
```

### `window.localStorage`

Documentation on `window.localStorage` can be found here: https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage.

```typescript
import proxyWithPersist from 'valtio-persist';
import type { ProxyPersistStorageEngine } from 'valtio-persist';

const storage: ProxyPersistStorageEngine = {
  getItem: name => window.localStorage.getItem(name),
  setItem: (name, value) => window.localStorage.setItem(name, value),
  removeItem: name => window.localStorage.removeItem(name),
  getAllKeys: () => Object.keys(window.localStorage);
};

const stateProxy = proxyWithPersist({
  getStorage: () => storage;
});
```

### `@react-native-async-storage/async-storage`

Documentation on `AsyncStorage` can be found here: https://github.com/react-native-async-storage/async-storage.

```
npm i @react-native-async-storage/async-storage
```

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import proxyWithPersist from 'valtio-persist';
import type { ProxyPersistStorageEngine } from 'valtio-persist';

const storage: ProxyPersistStorageEngine = {
  getItem: name => AsyncStorage.getItem(name),
  setItem: (name, value) => AsyncStorage.setItem(name, value),
  removeItem: name => AsyncStorage.removeItem(name),
  getAllKeys: () => AsyncStorage.getAllKeys();
};

const stateProxy = proxyWithPersist({
  getStorage: () => storage;
});
```

### `expo-file-system`

Documentation on `expo-file-system` can be found here: https://docs.expo.dev/versions/latest/sdk/filesystem.

```
expo install expo-file-system
```

```typescript
import * as FileSystem from 'expo-file-system';
import proxyWithPersist from 'valtio-persist';
import type { ProxyPersistStorageEngine } from 'valtio-persist';

const storage: ProxyPersistStorageEngine = {
  getItem: name => FileSystem.readAsStringAsync(FileSystem.documentDirectory + name),
  setItem: (name, value) => FileSystem.writeAsStringAsync(FileSystem.documentDirectory + name, value),
  removeItem: name => FileSystem.deleteAsync(FileSystem.documentDirectory + name),
  getAllKeys: () => FileSystem.readDirectoryAsync(FileSystem.documentDirectory);
};

const stateProxy = proxyWithPersist({
  getStorage: () => storage;
});
```

## Persist Strategies

There are two techniques to persist, "single file" (`PersistStrategy.SingleFile`) or "multi-file" (`PersistStrategy.MultiFile`).

### Single File

The single file strategy will stringify the value and store it into one file.

In the example here, any time a photo is added, or removed, or a key in the photo is updated, `JSON.stringify` runs on the entire `photos` object, and then this is written to file.

```typescript
const stateProxy = proxyWithPersist({
  // ...

  initialState: {
    photos: {
      1: { id: 1, views: 0 },
      2: { id: 2, views: 0 },
      3: { id: 3, views: 0 },
      4: { id: 4, views: 0 }
    }
  }

  persistStrategies: {
    photos: PersistStrategy.SingleFile
  }
})
```

### Multi-file

There is a second strategy called multi-file. This can only be used on keys that have an object-type value. Each key in the object will be turned into a file. This offers improved performance, because the entire value of of the object is not stringified, just individual values of the keys in the object are stringified, and then written to its own file.

In the example above, `photos` has an object-type value, so let's use multi-file strategy here.

```diff
const stateProxy = proxyWithPersist({
  // ...

  initialState: {
    photos: {
      1: { id: 1, views: 0 },
      2: { id: 2, views: 0 },
      3: { id: 3, views: 0 },
      4: { id: 4, views: 0 }
    }
  }

  persistStrategies: {
-    photos: PersistStrategy.SingleFile
+    photos: PersistStrategy.MultiFile
  }
})
```

Now adding a photo with key `5` and value of `{id: 5, views: 0 }` will only stringify this value and write it to disk. Updating the `photos['2'].views` to value of `99` will only stringify the photo at this position, and write it to it's individual file.

## Whitelisting

To only persist certain keys, define an object for the `persistStrategies` option. The keys of this object are dot path notation for the paths you want to store. Here is an example:

```typescript
const stateProxy = proxyWithPersist({
  // ...

  initialState: {
    entities: {
      tasks: {},
      schedules: {},
    },
  },

  persistStrategies: {
    'entities.tasks': PersistStrategy.SingleFile,
  },
});
```

In this example, only `stateProxy.entities.tasks` will get persisted. Any changes to `stateProxy.entities.schedules` or anywhere else, will not get persisted.

## Migrations

The two keys in the config argument of `proxyWithPersist` related to migrations are `version` and `migrations`.

The `version` is required and must be a number. Any time persisted data is loaded, it compares the persisted version, to the current version passed into `proxyWithPersist` argument. If the persisted version is less than the one passed in to the argument, `migrations` will then be run in ascending order of numbered key.

The `migrations` option must be an object where each key is a version. The value is an async function, it receives no arguments, and returns nothing, it just mutates the proxy object. All the migrations will be run that have a number key that is greater than persisted version and less-than-or-equal-to the `version` passed into `proxyWithPersist`.

Example:

The last persisted version was `0`.

```typescript
const stateProxy = proxyWithPersist({
  // ...

  version: 2,
  migrations: {
    1: async () => {
      stateProxy.counter = {};
    },

    2: async () => {
      delete stateProxy.foo;
    },
  },
});
```

When the app runs, it finds the last persisted version was `0`, but the current version is `2`. It will first run migration with key of `1` and then it will run migration with key of `2` and then `_persist.loaded` will be set to `true`.

## Recipes

### Throttle Writes for Performance

Sometimes, writing to disk on every change immediately hurts performance. Here is a technique to changes get persisted at most once a second. It uses the [`throttle`](https://lodash.com/docs/4.17.15#throttle) method from lodash. It will save to disk at most once a second.

Note: Debounce is not recommended as it could lead to starvation. For example, if writes are debounced to 1 second, but writes happen after 0.5s, then a write will never happen.

```
npm i lodash
```

```typescript
import { throttle } from 'lodash';

const stateProxy = proxyWithPersist({
  // ...

  onBeforeBulkWrite: throttle(bulkWrite => bulkWrite(), 1000)
}
```
