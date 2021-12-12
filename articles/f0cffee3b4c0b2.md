---
title: "[jest]DIしたオブジェクトをmockして、メソッドが呼ばれているかの確認をする"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "jest"]
published: true
---

## はじめに

DDDで実装していると、interfaceを定義してDIを行うことが多いと思います。

今回はその部分をmockして、関数が呼び出されたか？正しい引数で呼んでいるかの確認をしたいと思います


## テスト対象のコード

下記コードをテストします。
`UseCase`クラスでは`Repository`のinterfaceをDIしており、ユーザの存在有無で処理が分かれています。

このとき、insertとupdateが正しく呼ばれているかの確認を行おうと思います。

```ts:index.ts
export class UseCase {
  constructor(private repository: Repository) {}

  async execute(id: string, name: string, age: number): Promise<void> {
    const user = await this.repository.find(id);

    if (!user) {
      const newUser = {
        id,
        name,
        age,
      };
      await this.repository.insert(newUser);
      return;
    }

    await this.repository.update({
      id,
      name,
      age,
    });
  }
}
```

ちなみに`Repository`のinterfaceは下記のようなイメージです。

```ts:repository.ts
type User = {
  id: string;
  name: string;
  age: number;
};

// Repositoryのインターフェイス
export interface Repository {
  find(id: User['id']): Promise<User | undefined>;
  update(user: User): Promise<void>;
  insert(user: User): Promise<void>;
}

// 実装のイメージ
class dbRepository implements Repository {
  async find(id: User['id']) {
    console.log('dbからfind');
    return {
      id: '123',
      name: 'taro',
      age: 20,
    };
  }

  async update(user: User) {
    console.log('dbにupdate');
  }

  async insert(user: User) {
    console.log('dbにinsert');
  }
}
```


## テストコード

`Repository`のinterfaceを定義したmock関数を作成します。
また、テストの確認時では関数が呼ばれたかと引数が正しいかの確認を行いため、各メソッドのmockも返しておきます。

```ts:index.test.ts

// テストケース側からfindで返す値を決めたいので、userの引数を渡す
const userMockRepository = (user?: User) => {
  // find時は引数で与えたものを返す
  const findMock = jest.fn().mockReturnValue(Promise.resolve(user));
  const updateMock = jest.fn();
  const insertMock = jest.fn();

  // Repository interfaceを実装したmock関数を作成する
  const mockRepository = jest.fn<Repository, []>().mockImplementation(() => ({
    find: findMock,
    update: updateMock,
    insert: insertMock,
  }));

  return {
    mockRepository,
    findMock,
    updateMock,
    insertMock,
  };
};

// 実際のテストコード
describe('usecase', () => {
  it('新規ユーザの場合、追加される', async () => {
    // repositoryのmock関数を作成
    const repository = userMockRepository();
    const usecase = new UseCase(repository.mockRepository());

    await usecase.execute('newUser', 'taro', 20);

    // findが正しい引数で呼ばれたか
    expect(repository.findMock).toHaveBeenCalledWith('newUser');

    // insertとupdateが正しい回数で呼ばれたか
    expect(repository.insertMock).toBeCalled();
    expect(repository.updateMock).not.toBeCalled();

    // 下記のよう確認もできる
    // expect(repository.insertMock.mock.calls.length).toBe(1);
    // expect(repository.updateMock.mock.calls.length).toBe(0);
    
    // insertメソッドの引数が想定通りか
    expect(repository.insertMock).toHaveBeenCalledWith({ id: 'newUser', name: 'taro', age: 20 });
  });

  it('更新ユーザの場合、更新される', async () => {
    const repository = userMockRepository({ id: 'updateUser', name: 'name', age: 33 });
    const usecase = new UseCase(repository.mockRepository());

    await usecase.execute('updateUser', 'taro', 20);

    expect(repository.insertMock).not.toBeCalled();
    expect(repository.updateMock).toBeCalled();

    expect(repository.updateMock).toHaveBeenCalledWith({ id: 'updateUser', name: 'taro', age: 20 });
  });
});
```

### 参考

https://qiita.com/NeGI1009/items/e90033d1b2bc58a2766d