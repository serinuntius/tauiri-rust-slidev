---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# use UnoCSS (experimental)
# css: unocss
---

# TauriとRustとSolidJSで作るEthereum Address精製機
---

# What is Tauri?

TauriはRust製のアプリフレームワークです

- 🪶 **軽量** - アプリのサイズがElectron比で1/16です
- 💻 **省メモリ** - メモリ使用量がElectron比で 60% 少ないです
- ⚡️ **起動が速い** - 起動もめちゃんこ速いです
- 🔥 **No Chromium** - Chromiumに依存していません
- 🌈 **並列処理も得意** - バックエンドはRustなのでマルチスレッドも得意だよ
- 📱 **モバイルも対応予定** - iOSやAndroidやWASMにも対応予定です（soon）
- 🔑 **セキュア** - allowlistでAPIごとに詳細に権限を分けれるので安全です
---

# What is Ethereum Address Generator？
好きなEthereumのアドレスを精製するヤツ

---

# Demo

<a href="https://gyazo.com/c6151058daddb55f23306a76ea5d57f5"><video alt="Video from Gyazo" width="800" autoplay muted loop playsinline controls><source src="https://i.gyazo.com/c6151058daddb55f23306a76ea5d57f5.mp4" type="video/mp4" /></video></a>

---

# What is SolidJS ?

- ⚡ **ハイパフォーマンス** - UIのスピードとメモリ使用率のベンチで常にトップらしい
- 💻 **Reactっぽい** - JSXで書ける！ `useState` 的な概念もだいたい似てる
- ㌰㌍️ **簡単** - Reactのhooksは初心者にはとっつき辛いルールの上で成り立っているが、SolidJSはその辺をカバーしている感じがする

ちなみにTauriは別にSolidJSは **マストではない！** React, Vue, Vanilla等々なんでも書ける。
---

# コードの解説とか(Tauri側)
[view full page](https://github.com/serinuntius/ethereum-address-generator)

```rust
#[tauri::command(async)]
async fn generate_address(start_with: String) -> Result<(String, String), ()> {
  ...省略
  let key = Wallet::<SigningKey>::new(&mut rand::thread_rng());
  let address = key.address().to_string();

  if address.starts_with(start_with.as_str()) {
    let hex_key = key.signer().to_bytes().encode_hex();
    println!("hex_key: {}", hex_key);
    if let Some(err) = tx.send((format!("{:?}", key.address()), hex_key)).err() {
      println!("err: {}", err);
    }
    break;
  }
  ...省略

  let answer = rx.recv().expect("failed to recvs");

  Ok(answer)
}

```

--- 

# コードの解説とか(SolidJS側)

```tsx
const calcByRust = async  (target: string): Promise<string[]> =>  {
    // await invoke('関数名')ってやるだけで非同期でRustの関数呼び出せる！
    // お手軽！！！ 
    const result = await invoke('generate_address', {startWith: target}) as string[];
    return [result[0],`0x${result[1]}`];
}
```

Reactと違うのは
```tsx
// `useState` の代わりに `createSignal` を使う
const [address, setAddress] = createSignal<string>('');
const [secretKey, setSecretKey] = createSignal<string>('');
```

```tsx
// しかも `createSignal` はどこにでも置ける
const [address, setAddress] = createSignal<string>('');
const Component = () => {
    return (
        <div>
          {address()}
        </div>
    );
}
```

---

# 仕事で実際に使ってみてわかったRustのいいところ
## Result型がいい！
あえて冗長に書いてます。
```rust
fn get_file(path: Path) -> Result<(), Error> {
  // 戻り値がResult型で、OkかErrが入ってる
  let file = File::open(path);

  let mut contents = String::new();
  match file {
    Ok(f) => f.read_to_string(&mut contents).unwrap(),
    Err(err) => panic!("panic!!! {:?}", err),
  }
  Ok(())
}
```
---

# Result型
- Result<T,E>は失敗するかもしれない処理の結果を表現する列挙型です。
- Resultを無視するとwarningが出るのでエラーハンドリングを強制&可視化できる！
- Rustには例外がない
```rust
let ok_value: Result<usize, &'static str> = Ok(1);
match ok_value {
    Ok(v) => println!("ok value = {}", v),
    Err(e) => println!("err value = {}", e),
};

// どちらか片方の場合だけ処理したいならif letが便利
if let Ok(v) = ok_value {
    println!("ok value = {}", v);
}

// Okの場合は中身を返し、Errの場合はpanicする
assert_eq!(ok_value.unwrap(), 1);

// unwrapと同じだが、panic時のエラーメッセージを指定できる
assert_eq!(ok_value.expect("panic"), 1);
```
エラーハンドリングをサボると可視化される仕組みがすごい
---

# ?でエラーを返す
`? 演算子` というシンタックスシュガーがあり、エラーをそのまま返せる
```rust

fn get_hoge() -> Result<(), Error> {
  let err_value: Result<usize, &'static str> = Err("error message");
  // errがある場合はerrをreturnする, no_errにはOk時のあたいが入る
  let no_err = err_value?;
  
  // errの場合はここにはもう到達しない
}
```
Goが嫌いって言う人がよく言う `if err != nil { return err }` を書かなくていいw
```go
// こんなやつ
if err := get_hoge(); err != nil {
  return nil, err;
}
```
間違いなくスマートではある。ちなみにそんなに気にならない派。
---

# コンパイラがめちゃくちゃ賢い
- なんで君それがやばいってわかるの？ってレベルで丁寧に検査されている
- 最初はコンパイル通すのが大変だけど、通った時の安心感はやばい
- 100%バグなく正しく動きそうな気がするw(決してそんなことはないけど、それぐらい型付けもしっかりしてる)

---

# 学習曲線云々は、間違いなく誇大広告だと思う
- いざ仕事で書かなきゃいけない状況に追い込まれると、書けるようになる（経験談
- コンパイラエラーは、慣れるとすぐに解決できるようになるし、コンパイラのお気持ちもわかってくる
- Rust書いたこともないのに、学習コストが〜〜〜とか言ってる **エアプおじさん** は無視して🆗
- あと書く環境が大事
  - 特にこだわりがなければ、VS Codeで `rust-analyzer` 入れれば神
- 静的型付け言語やったことがあって、参照とかのスタックとヒープの話がわかってればOK
- 型推論も効くし最高
---

# まとめ
- Rust最高
- Tauri最高
- SolidJS最高
---

# 参考文献

- [RustのOptionとResult](https://qiita.com/take4s5i/items/c890fa66db3f71f41ce7#resultte%E5%9E%8B)
- [Rust GUI の決定版！ Tauri を使ってクロスプラットフォームなデスクトップアプリを作ろう](https://zenn.dev/kumassy/books/6e518fe09a86b2)
