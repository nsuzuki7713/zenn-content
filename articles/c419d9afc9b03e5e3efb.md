---
title: "【GCP】Cloud StorageをNodeのSDKから実行してみる"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GCS]
published: true
---

## はじめに

SDKを介してCloud Storageを色々触ってみたので、その時にコードを残しておきます。
環境はローカルから、nodeのSDKを使用しています。

```
$ node -v
v15.11.0

"@google-cloud/storage": "^5.8.1"
```

## バケットの操作

### バケットの作成

バケットの作成は`Storage.createBucket`を使用します。

https://googleapis.dev/nodejs/storage/latest/Storage.html#createBucket

```ts:createBuncket.ts
import { CreateBucketRequest, Storage } from '@google-cloud/storage';
import { v4 } from 'uuid';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucketName = `createbucket-${v4()}`;
  const metadata: CreateBucketRequest = {
    location: 'asia-northeast1',
    storageClass: 'standard',
  };
  const [bucket] = await storage.createBucket(bucketName, metadata);
  console.log(`${bucket.name}`);
})().catch(async (e) => {
  console.log(e);
});
```

すべてのバケット名が一意である必要があるため、例えば`test`というバケット(既に他の誰かが作成したバケット)を作成するとエラーとなります。
```
409 [
  {
    message: 'Sorry, that name is not available. Please try a different one.',
    domain: 'global',
    reason: 'conflict'
  }
]
```

また、自分が既に作成したバケット名を指定するとエラーメッセージが変わります。
```
409 [
  {
    message: 'You already own this bucket. Please select another name.',
    domain: 'global',
    reason: 'conflict'
  }
]
```

エラーレスポンスの一覧がどこにあるのか、いまいち分かっていないのですが下記が参考になるでしょうか。
https://cloud.google.com/storage/docs/xml-api/reference-status?hl=ja


また、バケットの作成は`Bucket.create`を使用することも可能です。
https://googleapis.dev/nodejs/storage/latest/Bucket.html#create

```ts:createBucket2
import { Storage } from '@google-cloud/storage';
import { v4 } from 'uuid';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucketName = `createbucket-${v4()}`;
  const options = {
    location: 'asia-northeast1',
    storageClass: 'standard',
  };
  const bucket = storage.bucket(bucketName);
  await bucket.create(options);
  console.log(bucketName);
})().catch(async (e) => {
  console.log(e);
  console.log(e.code, e.errors);
});
```

### バケットの一覧

バケットの一覧は`Storage.getBuckets`で使用できます。

https://googleapis.dev/nodejs/storage/latest/Storage.html#getBuckets

```ts:getBuckets.ts
import { GetBucketsRequest, Storage } from '@google-cloud/storage';

(async () => {
　const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const options: GetBucketsRequest = {
    // prefixをオプションで指定できる
    // この場合は`createbucket-`から始まるバケット名を取得できる
    prefix: 'createbucket-',
  };
  const [buckets] = await storage.getBuckets(options);
  buckets.forEach((bucket) => console.log(bucket.name));
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

実行結果
```
createbucket-14de618e-a92e-465f-bd5d-2c83e82160ef
createbucket-370c1b94-33ed-428f-8a93-dcded81e7f46
```

### バケットの削除

バケッドの削除は`Bucket.delete`を使用できます。

https://googleapis.dev/nodejs/storage/latest/Bucket.html#delete

```ts:deleteBucket.ts
import { Storage, DeleteBucketOptions } from '@google-cloud/storage';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-ec2f9f4f-c2e2-4a04-9959-b911e83a7f89');

  const options: DeleteBucketOptions = {
    // バケットが存在しない場合に例外を投げない
    ignoreNotFound: true,
  };
  await bucket.delete(options);
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

`ignoreNotFound`を`false`にして、バケットが存在しない場合は、
`404 [ { message: 'Not Found', domain: 'global', reason: 'notFound' } ]` のエラーを発生します。

また、バケットの削除はバケット内のオブジェクトがすべて削除されている必要があり、オブジェクトが存在する場合は削除できません。

オブジェクトが存在する場合のエラー内容
```
409 [
  {
    message: 'The bucket you tried to delete was not empty.',
    domain: 'global',
    reason: 'conflict'
  }
]
```

pythonのクライアントではオブジェクトの削除も実施してくれるオプションがありましたが、nodeのではそういうのは無さそうでした。
そのため、事前にオブジェクト削除する必要があります。

https://googleapis.dev/python/storage/latest/buckets.html#google.cloud.storage.bucket.Bucket.delete

## オブジェクトの操作

### オブジェクトのアップロード

オブジェクトのアップロートは`Bucket.upload`を使用します。

https://googleapis.dev/nodejs/storage/latest/Bucket.html#upload

```ts:upload.ts
import { Storage, UploadOptions } from '@google-cloud/storage';
import { v4 } from 'uuid';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-69fd9e10-25eb-4311-aac3-9641fa49c247');
  const options: UploadOptions = {
    contentType: 'text/plain',
    // 保存されるオブジェクト名
    destination: `${v4()}/uploadFile.txt`,
    // ファイルを自動的にgzip圧縮するか
    gzip: true,
  };

  await bucket.upload('./src/cloudStorage/object/uploadFile.txt', options);
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

複数のオブジェクトをuploadする場合はzipにまとめることで可能です。
```ts:upload2.ts
import { Storage, UploadOptions } from '@google-cloud/storage';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-69fd9e10-25eb-4311-aac3-9641fa49c247');
  const options: UploadOptions = {
    contentType: 'application/zip',
  };

  await bucket.upload('./src/cloudStorage/object/uploadObject.zip', options);
})().catch(async (e) => {
  console.log(e);
  console.log(e.code, e.errors);
});
```

### オブジェクトのダウンロード

オブジェクトのダウンロードは`File.download`を使用します。

https://googleapis.dev/nodejs/storage/latest/File.html#download

```ts:download.ts
import { Storage, DownloadOptions } from '@google-cloud/storage';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-69fd9e10-25eb-4311-aac3-9641fa49c247');
  const srcFileName = '35178b3c-602d-44db-a01a-92f995ac3165/uploadFile';
  const options: DownloadOptions = {
    // ダウンロードされるローカルのPath
    destination: './download.txt',
  };

  await bucket.file(srcFileName).download(options);
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

### オブジェクトの一覧取得

このようなオブジェクトが存在する前提で実行結果を載せています。

```
a.txt
a/b/c/d.txt
b/d/e.txt
b/d/f.txt
b/d/g/h.txt
```

オブジェクトの一覧は`Bucket.getFiles`を使用します。

https://googleapis.dev/nodejs/storage/latest/Bucket.html#getFiles

```ts
import { Storage, GetFilesOptions } from '@google-cloud/storage';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-69fd9e10-25eb-4311-aac3-9641fa49c247');

  const options: GetFilesOptions = {};
  const [files] = await bucket.getFiles(options);

  files.forEach((file) => {
    console.log(file.name);
  });
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

実行結果
```
a.txt
a/
a/b/
a/b/c/
a/b/c/d.txt
b/
b/d/
b/d/e.txt
b/d/f.txt
b/d/g/
b/d/g/h.txt
```

`delimiter`と`prefix`を指定することでオブジェクトを絞り込むことが可能です。
```ts
import { Storage, GetFilesOptions } from '@google-cloud/storage';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-3f8018e7-76c8-4aa8-8613-47b218feb1a3');

  const options: GetFilesOptions = {
    autoPaginate: true,
    delimiter: '/',
    prefix: 'b/d/',
  };

  const files = await bucket.getFiles(options);
  files[0].forEach((file) => {
    console.log(file.name);
  });

  files[2].prefixes.forEach((dir: any) => {
    console.log(dir);
  });
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

実行結果
```
b/d/
b/d/e.txt
b/d/f.txt
```

サブディレクトリの対応はこちらの記事が参考になりました。
https://www.kwbtblog.com/entry/2020/01/30/030605

### オブジェクトの削除

オブジェクトの削除は`File.delete`を使用します。

https://googleapis.dev/nodejs/storage/latest/File.html#delete

```ts
import { Storage, DeleteFileOptions } from '@google-cloud/storage';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-69fd9e10-25eb-4311-aac3-9641fa49c247');
  const srcFileName = 'a.txt';
  const options: DeleteFileOptions = {
    // オブジェクトが存在しない場合エラーにするかどうか
    ignoreNotFound: true,
  };

  await bucket.file(srcFileName).delete(options);
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

また、一括で複数のオブジェクト使用する場合は、`Bucket.deleteFiles`を使用することができます。
https://googleapis.dev/nodejs/storage/latest/Bucket.html#deleteFiles

このメソッドはatomicなリクエストではないので、一部が削除されないパターンもあるので注意が必要です。

```ts
import { Storage, DeleteFilesOptions } from '@google-cloud/storage';

(async () => {
  const storage = new Storage({projectId: 'user-projectId', keyFilename: './key.json'});

  const bucket = storage.bucket('createbucket-69fd9e10-25eb-4311-aac3-9641fa49c247');
  const options: DeleteFilesOptions = {
    // getFilesと同じパラメータを使用できるため、対象のオブジェクトを絞り込むことが可能

    // trueの場合はすべてのファイルが処理されるまでエラーにしない
    // flaseの場合は最初にエラーになったら、例外を投げる
    force: true,
  };

  await bucket.deleteFiles(options);
})().catch(async (e) => {
  console.log(e.code, e.errors);
});
```

## clound functionのトリガー

Cloud Storageではオブジェクトの操作をした際に通知をすることができます。
ここではclound functionのイベントを確認します。

コンソールから`Cloud Functionsで処理`をクリック

![](https://storage.googleapis.com/zenn-user-upload/i52kzpj201rvbg81gqq8ybyzsocr)

Cloud Functionの作成
![](https://storage.googleapis.com/zenn-user-upload/g2rgtb51vtpumss1fg0ntq6ifx30)


トリガーの種類は以下となります。１つのfunctionには１つのトリガーしか設定できないようです。
https://cloud.google.com/functions/docs/tutorials/storage?hl=ja#object_finalize


動作確認用のため、ログを仕込みます
```js:index.js
exports.helloGCS = (file, context) => {
  console.log(`  Event: ${context.eventId}`);
  console.log(`  Event Type: ${context.eventType}`);
  console.log(`  Bucket: ${file.bucket}`);
  console.log(`  File: ${file.name}`);
  console.log(`  Metageneration: ${file.metageneration}`);
  console.log(`  Created: ${file.timeCreated}`);
  console.log(`  Updated: ${file.updated}`);
};
```

![](https://storage.googleapis.com/zenn-user-upload/mikqmcjjdeh1glgbdpt8jz7eh1mb)


オブジェクトをuploadするとイベントが実行されていことが分かります。
![](https://storage.googleapis.com/zenn-user-upload/6wl87kzzl1b7gdb6bbhfmx3g0bp7)

## おわりに

CRUDの処理をざっくり触ってみました。他にもPubSubやstream処理は使う機会がありそうなので、またの機会に確認して追加しようと思います。