# ファイルのアップロード

おそらく聞いたことがあると思いますが、Redwoodは未来はサーバーレスだと考えています。この概念は、これまで心配する必要がなかったかもしれないいくつかの興味深い問題をもたらします。例：ファイルをアップロードすると、ファイルはどこに移動しますか。サーバーはありません。過去に[自分](https://redwoodjs.com/tutorial/authentication)で行った可能性のある多くのタスクと同様に、これはサードパーティのサービスに委託できるもう1つの仕事です。

## サービス

CDNからのファイルのアップロードと提供を処理するサービスはたくさんあります。大きなものの2つは、 [Cloudinary](https://cloudinary.com)と[Filestack](https://filestack.com)です。統合が非常に簡単であることがわかったので、ここでFilestack統合のデモを行います。アップロードを保存してCDN経由で利用できるようにするだけでなく、オンザフライの画像変換も提供するため、誰かがRetina対応の5000ピクセル幅のヘッドショットをアップロードした場合でも、アップロードを縮小して、アバターに100ピクセルバージョンのみを提供できます。サイトの右上隅にあります。帯域幅と転送コストを節約できます。

月に100回のアップロード、1000回の変換（画像のサイズ変更など）、1 GBの帯域幅、0.5GBのストレージを提供する無料プランにサインアップします。このデモ、そしておそらくトラフィックの少ない本番サイトには、これで十分です。

https://dev.filestack.com/signup/free/にアクセスして、サインアップしてください。ログインする前に確認メールが送信されるため、必ず実際のメールアドレスを使用してください。メールを確認すると、ダッシュボードにドロップされ、APIキーが右上に表示されます。

![新しい画像の足場](https://user-images.githubusercontent.com/300/82616735-ec41a400-9b82-11ea-9566-f96089e35e52.png)

それをコピーするか、少なくともブラウザタブを開いたままにしておきます。これは、すぐに必要になるためです。 （私はすでにそのキーを変更したので、それを盗もうとしないでください！）

ファイルスタック側、アプリケーションについては以上です。

## アプリ

ユーザーが画像をアップロードしてカタログ化できる非常に単純なDAM（Digital Asset Manager）を作成しましょう。サムネイルをクリックすると、フルサイズのバージョンを開くことができます。

新しいRedwoodアプリを作成します。

```terminal
yarn create redwood-app uploader
cd uploader
```

最初に行うことは、FilestackAPIキーを保持するための環境変数を作成することです。これは、詮索好きな目で見るためにキーがリポジトリに存在しないようにするためのベストプラクティスです。アプリのルートにある`.env`ファイルにキーを追加します。

```terminal
REDWOOD_ENV_FILESTACK_API_KEY=AM18i8xV4QpoiGwetoTWd
```

> We're prefixing with `REDWOOD_ENV_` here as an indicator to webpack that we want it to replace these variables with the actual values as it is processing pages and statically generating them. Otherwise our generated pages would still contain something like `process.env.FILESTACK_API_KEY`, which would not exist when the pages are static and being served from a CDN.

これで、開発サーバーを起動できます。

```terminal
yarn rw dev
```

### データベース

画像データを保存する単一のモデルを作成します。

```javascript
// api/prisma/schema.prisma

model Image {
  id    Int    @default(autoincrement()) @id
  title String
  url   String
}
```

`title`はこのアセットのユーザー指定の名前になり、 `url`にはFilestackがアップロード後に作成するパブリックURLが含まれます。

移行を作成し、データベースを更新します。

```terminal
yarn rw db save
yarn rw db up
```

私たちの生活を楽にするために、画像を作成/編集/削除するために必要な画面の足場を作りましょう。それらを変更してアップローダーを追加します。

```terminal
yarn rw g scaffold image
```

次に、http：// localhost：8910 / images / newにアクセスして、画像アップローダーを追加するために何をする必要があるかを考えてみましょう。

![新しい画像の足場](https://user-images.githubusercontent.com/300/82694608-653f0b00-9c18-11ea-8003-4dc4aeac7b86.png)

## アップローダー

Filestackには、すべてのアップロードを処理する[Reactコンポーネント](https://github.com/filestack/filestack-react)があります。パッケージを追加しましょう：

```terminal
yarn workspace web add filestack-react
```

スキャフォールドフォームにアップローダーが必要になることがわかっているので、それをインポートして、 **Url**入力をAPIキーを指定して置き換えてみましょう。

```javascript{11,54}
// web/src/components/ImageForm/ImageForm.js

import {
  Form,
  FormError,
  FieldError,
  Label,
  TextField,
  Submit,
} from '@redwoodjs/forms'
import ReactFilestack from 'filestack-react'

const CSS = {
  label: 'block mt-6 text-gray-700 font-semibold',
  labelError: 'block mt-6 font-semibold text-red-700',
  input:
    'block mt-2 w-full p-2 border border-gray-300 text-gray-700 rounded focus:outline-none focus:border-gray-500',
  inputError:
    'block mt-2 w-full p-2 border border-red-700 text-red-900 rounded focus:outline-none',
  errorMessage: 'block mt-1 font-semibold uppercase text-xs text-red-700',
}

const ImageForm = (props) => {
  const onSubmit = (data) => {
    props.onSave(data, props?.image?.id)
  }

  return (
    <div className="box-border text-sm -mt-4">
      <Form onSubmit={onSubmit} error={props.error}>
        <FormError
          error={props.error}
          wrapperClassName="p-4 bg-red-100 text-red-700 border border-red-300 rounded mt-4 mb-4"
          titleClassName="mt-0 font-semibold"
          listClassName="mt-2 list-disc list-inside"
        />

        <Label
          name="title"
          className={CSS.label}
          errorClassName={CSS.labelError}
        >
          Title
        </Label>
        <TextField
          name="title"
          defaultValue={props.image?.title}
          className={CSS.input}
          errorClassName={CSS.inputError}
          validation={{ required: true }}
        />
        <FieldError name="title" className={CSS.errorMessage} />

        <ReactFilestack apikey={process.env.REDWOOD_ENV_FILESTACK_API_KEY} />

        <div className="mt-8 text-center">
          <Submit
            disabled={props.loading}
            className="bg-blue-600 text-white hover:bg-blue-700 text-xs rounded px-4 py-2 uppercase font-semibold tracking-wide"
          >
            Save
          </Submit>
        </div>
      </Form>
    </div>
  )
}

export default ImageForm
```

よく見ると、タイトル入力の下に*小さな*ボタンがあります。

![ファイルを選択ボタン](https://user-images.githubusercontent.com/300/82617171-1c3d7700-9b84-11ea-9e70-d005c419ebe1.png)

これをクリックすると、ローカルファイルの選択、URLの提供、Facebook、Instagram、Googleドライブからの取得など、あらゆる種類のオプションを備えたピッカーが実際に起動します。悪くない！

![ファイルスタックピッカー](https://user-images.githubusercontent.com/300/82617240-51e26000-9b84-11ea-8aec-210b7a751e8c.png)

ユーザーにそのボタンをクリックさせる理由はありません。いくつかの[オプションを](https://github.com/filestack/filestack-react#props)追加して、ページが読み込まれたときにピッカーをページに表示してみましょう。コンテナを配置するためのコンテナを作成する必要があるため、 `<div>`を追加し、 `<ReactFilestack>`通知する`id`属性を指定します。また、ピッカーが0pxの高さに折りたたまれないように、 `<div>`にいくつかのスタイルを指定します。

```javascript
// web/src/components/ImageForm/ImageForm.js

<ReactFilestack
  apikey={process.env.REDWOOD_ENV_FILESTACK_API_KEY}
  componentDisplayMode={{ type: 'immediate' }}
  actionOptions={{ displayMode: 'inline', container: 'picker' }}
/>
<div id="picker" style={{ marginTop: '2rem', height: '20rem' }}></div>
```

すごい！画像をアップロードして、機能することを確認することもできます。

![アップロード](https://user-images.githubusercontent.com/300/82618035-bb636e00-9b86-11ea-9401-61b8c989f43c.png)

> ファイルを選択した後に表示される[**アップロード**]ボタンをクリックしてください。

Filestackダッシュボードに移動すると、画像がアップロードされていることがわかります。

![ファイルスタックダッシュボード](https://user-images.githubusercontent.com/300/82618057-ccac7a80-9b86-11ea-9cd8-7a9e80a5a20f.png)

しかし、それはデータベースレコードに何かを添付するのに役立ちません。そうしよう。

## データ

アップロードが完了したときに何が起こっているかを見てみましょう。 Filestackピッカーは、完了時に呼び出す関数を持つ`onSuccess`属性を取ります。

```javascript{10-12,18}
// web/src/components/ImageForm/ImageForm.js

// imports and stuff...

const ImageForm = (props) => {
  const onSubmit = (data) => {
    props.onSave(data, props?.image?.id)
  }

  const onFileUpload = (response) => {
    console.info(response)
  }

  // form stuff...

  <ReactFilestack
    apikey={process.env.REDWOOD_ENV_FILESTACK_API_KEY}
    onSuccess={onFileUpload}
    componentDisplayMode={{ type: 'immediate' }}
    actionOptions={{ displayMode: 'inline', container: 'picker' }}
  />
```

ここでよくlookie：

![アップローダーの応答](https://user-images.githubusercontent.com/300/82618071-ddf58700-9b86-11ea-9626-e093b4c8d853.png)

`filesUploaded[0].url`は、まさに必要なもののようです。アップロードされたばかりの画像へのパブリックURLです。優れた！フォームを送信するときに利用できるように、小さな状態を使用して追跡するのはどうですか。

```javascript{12,25,32}
// web/src/components/ImageForm/ImageForm.js

import {
  Form,
  FormError,
  FieldError,
  Label,
  TextField,
  Submit,
} from '@redwoodjs/forms'
import ReactFilestack from 'filestack-react'
import { useState } from 'react'

const CSS = {
  label: 'block mt-6 text-gray-700 font-semibold',
  labelError: 'block mt-6 font-semibold text-red-700',
  input:
    'block mt-2 w-full p-2 border border-gray-300 text-gray-700 rounded focus:outline-none focus:border-gray-500',
  inputError:
    'block mt-2 w-full p-2 border border-red-700 text-red-900 rounded focus:outline-none',
  errorMessage: 'block mt-1 font-semibold uppercase text-xs text-red-700',
}

const ImageForm = (props) => {
  const [url, setUrl] = useState(props?.image?.url)

  const onSubmit = (data) => {
    props.onSave(data, props?.image?.id)
  }

  const onFileUpload = (response) => {
    setUrl(response.filesUploaded[0].url)
  }

  return (
  // component stuff...

```

So we'll use `setState` to store the URL for the image. We default it to the existing `url` value, if it exists—remember that scaffolds use this same form for editing of existing records, where we'll already have a value for `url`. If we didn't store that url value somewhere then it would be overridden with `null` if we started editing an existing record!

最後に行う必要があるのは、 `onSave`ハンドラーに送信される前に`data`オブジェクトに`url`の値を設定することです。

```javascript{4,5}
// web/src/components/ImageForm/ImageForm.js

const onSubmit = (data) => {
  const dataWithUrl = Object.assign(data, { url })
  props.onSave(dataWithUrl, props?.image?.id)
}
```

次に、ファイルをアップロードしてからフォームを保存してみてください。

![アップロードが完了しました](https://user-images.githubusercontent.com/300/82702493-f5844c80-9c26-11ea-8fc4-0273b92034e4.png)

機能した！次に、ここで表示を更新して、実際に画像をサムネイルとして表示し、クリックしてフルバージョンを表示できるようにします。

```javascript{61-63}
// web/src/components/Images/Images.js

import { useMutation } from '@redwoodjs/web'
import { Link, routes } from '@redwoodjs/router'

const DELETE_IMAGE_MUTATION = gql`
  mutation DeleteImageMutation($id: Int!) {
    deleteImage(id: $id) {
      id
    }
  }
`

const MAX_STRING_LENGTH = 150

const truncate = (text) => {
  let output = text
  if (text && text.length > MAX_STRING_LENGTH) {
    output = output.substring(0, MAX_STRING_LENGTH) + '...'
  }
  return output
}

const timeTag = (datetime) => {
  return (
    <time dateTime={datetime} title={datetime}>
      {new Date(datetime).toUTCString()}
    </time>
  )
}

const ImagesList = ({ images }) => {
  const [deleteImage] = useMutation(DELETE_IMAGE_MUTATION)

  const onDeleteClick = (id) => {
    if (confirm('Are you sure you want to delete image ' + id + '?')) {
      deleteImage({ variables: { id }, refetchQueries: ['IMAGES'] })
    }
  }

  return (
    <div className="bg-white text-gray-900 border rounded-lg overflow-x-scroll">
      <table className="table-auto w-full min-w-3xl text-sm">
        <thead>
          <tr className="bg-gray-300 text-gray-700">
            <th className="font-semibold text-left p-3">id</th>
            <th className="font-semibold text-left p-3">title</th>
            <th className="font-semibold text-left p-3">file</th>
            <th className="font-semibold text-left p-3"> </th>
          </tr>
        </thead>
        <tbody>
          {images.map((image) => (
            <tr
              key={image.id}
              className="odd:bg-gray-100 even:bg-white border-t"
            >
              <td className="p-3">{truncate(image.id)}</td>
              <td className="p-3">{truncate(image.title)}</td>
              <td className="p-3">
                <a href={image.url} target="_blank">
                  <img src={image.url} style={{ maxWidth: '50px' }} />
                </a>
              </td>
              <td className="p-3 pr-4 text-right whitespace-no-wrap">
                <nav>
                  <ul>
                    <li className="inline-block">
                      <Link
                        to={routes.image({ id: image.id })}
                        title={'Show image ' + image.id + ' detail'}
                        className="text-xs bg-gray-100 text-gray-600 hover:bg-gray-600 hover:text-white rounded-sm px-2 py-1 uppercase font-semibold tracking-wide"
                      >
                        Show
                      </Link>
                    </li>
                    <li className="inline-block">
                      <Link
                        to={routes.editImage({ id: image.id })}
                        title={'Edit image ' + image.id}
                        className="text-xs bg-gray-100 text-blue-600 hover:bg-blue-600 hover:text-white rounded-sm px-2 py-1 uppercase font-semibold tracking-wide"
                      >
                        Edit
                      </Link>
                    </li>
                    <li className="inline-block">
                      <a
                        href="#"
                        title={'Delete image ' + image.id}
                        className="text-xs bg-gray-100 text-red-600 hover:bg-red-600 hover:text-white rounded-sm px-2 py-1 uppercase font-semibold tracking-wide"
                        onClick={() => onDeleteClick(image.id)}
                      >
                        Delete
                      </a>
                    </li>
                  </ul>
                </nav>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}

export default ImagesList
```

![画像](https://user-images.githubusercontent.com/300/82702575-1fd60a00-9c27-11ea-8d2f-047bcf4e9cae.png)

## 変換

Filestackがその場で画像を変換することで帯域幅を節約できると言ったときのことを覚えていますか？このページは完璧な例です。画像が50pxを超えることはありません。なぜ、その小さなディスプレイのためだけにフル解像度をプルダウンするのでしょうか。これが、画像のこのインスタンスを取得するときは常に100pxで十分であることをFilestackに伝える方法です。なぜ100px？現在、ほとんどの電話と多くのラップトップおよびデスクトップディスプレイは4k以上です。これらのディスプレイでは、実際には画像が少なくとも2倍の解像度で表示されるため、「50ピクセル」であっても、これらのディスプレイに表示すると実際には100ピクセルになります。したがって、通常は、意図した表示解像度の2倍ですべての画像を停止する必要があります。

変換をトリガーするためにURL自体に特別なインジケーターを追加する必要があるので、特定の画像URLに対してそれを行う関数を追加しましょう（これはコンポーネント定義の内部または外部に配置できます）。

```javascript
// web/src/components/Images/Images.js

const thumbnail = (url) => {
  const parts = url.split('/')
  parts.splice(3, 0, 'resize=width:100')
  return parts.join('/')
}
```

What this does is turn a URL like `https://cdn.filestackcontent.com/81m7qIrURxSp7WHcft9a` into `https://cdn.filestackcontent.com/resize=width:100/81m7qIrURxSp7WHcft9a`.

次に、その関数の結果を`<img>`タグで使用します。

```javascript
// web/src/components/Images/Images.js

<img src={thumbnail(image.url)} style={{ maxWidth: '50px' }} />
```

アップロードされた157kBの画像から始めて、100pxのサムネイルはわずか6.5kBでクロックインします！画像配信を最適化することは、ほとんどの場合、余分な努力の価値があります！

You can read more about the available transforms over at [Filestack's API reference](https://www.filestack.com/docs/api/processing/).

## 改善点

アップロードした後、アップロードした画像が表示されると便利です。同様に、画像を編集するときは、すでに添付されているものを確認すると便利です。今、それらの改善を行いましょう。

添付画像のURLはすでに状態で保存されているので、その状態の存在を利用して添付画像を表示してみましょう。実際、アップローダーを非表示にして、完了したと仮定しましょう（必要に応じて再度表示できるようになります）。

```javascript{14,18}
// web/src/components/ImageForm/ImageForm.js

<ReactFilestack
  apikey={process.env.REDWOOD_ENV_FILESTACK_API_KEY}
  onSuccess={onFileUpload}
  componentDisplayMode={{ type: 'immediate' }}
  actionOptions={{ displayMode: 'inline', container: 'picker' }}
/>
<div
  id="picker"
  style={{
    marginTop: '2rem',
    height: '20rem',
    display: url ? 'none' : 'block',
  }}
></div>

{url && <img src={url} style={{ marginTop: '2rem' }} />}
```

これで、新しい画像を作成するとピッカーが表示され、アップロードが完了するとすぐに、アップロードされた画像が所定の位置に表示されます。画像を編集しようとすると、すでに添付されているファイルが表示されます。

> ここではおそらく同じサイズ変更URLトリックを使用する必要があるため、アップロード直後に10MBの画像を表示しようとしないようにしてください。 500pxの最大幅が良いかもしれません...

次に、画像を変更する場合にアップローダーを戻す機能を追加しましょう。状態にある画像をクリアすることでそれを行うことができます：

```javascript{18-29}
// web/src/components/ImageForm/ImageForm.js

<ReactFilestack
  apikey={process.env.REDWOOD_ENV_FILESTACK_API_KEY}
  onSuccess={onFileUpload}
  componentDisplayMode={{ type: 'immediate' }}
  actionOptions={{ displayMode: 'inline', container: 'picker' }}
/>
<div
  id="picker"
  style={{
    marginTop: '2rem',
    height: '20rem',
    display: url ? 'none' : 'block',
  }}
></div>

{url && (
  <div>
    <img src={url} style={{ display: 'block', margin: '2rem 0' }} />
    <a
      href="#"
      onClick={() => setUrl(null)}
      className="bg-blue-600 text-white hover:bg-blue-700 text-xs rounded px-4 py-2 uppercase font-semibold tracking-wide"
    >
      Replace Image
    </a>
  </div>
)}
```

![画像ボタンを置き換える](https://user-images.githubusercontent.com/300/82719274-e7055780-9c5d-11ea-9a8a-8c1c72185983.png)

送信ボタンからスタイルを借用し、画像に上下の余白があることを確認して、新しいボタンにクラッシュしないようにしました。

## まとめ

アップロードされたファイル！ファイルピッカーを統合する方法はたくさんあり、これは1つにすぎませんが、シンプルでありながら柔軟性があると考えています。 [example-blog](https://github.com/redwoodjs/example-blog)でも同じ手法を使用します。

楽しんでアップロードしてください！
