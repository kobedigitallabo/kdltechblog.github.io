---
layout: post
title: generator-react-webpack+Material UIでMaterial UIなWebアプリをさくっと開発する
published: true
tags: [material]
---

村岡です。個人的にWebアプリは[Yeoman](http://yeoman.io/)で作りはじめることがけっこう多いです。開発環境のセットアップとか自動化とか複雑なことをいろいろ考えなくて済むのがいいですね。  
とくに最近[generator-react-webpack](https://github.com/newtriks/generator-react-webpack)での開発がなかなか便利だと思ってます。あとReactの[Material UI](http://www.material-ui.com/)コンポーネントがいい出来なのでこれらを組み合わせてみようと思います。

# セットアップ

[Yeoman](http://yeoman.io/)とgenerator-react-webpackのインストールがまだの場合はインストールします。

```
node -v
6.4.0
npm install -g yo
npm install -g generator-react-webpack
```

## プロジェクトを作成する

```
mkdir my-project && cd $_
yo react-webpack
...
? Please choose your application name (my-project) my-project
? Which style language do you want to use? (Use arrow keys)
❯ css 
? Enable postcss? (y/N) N
```

上記を実行するとモジュールのインストール等が実行されます。少し時間がかかります。

# 開発用Webサーバーの起動

インストールが終わったら `npm start`でいったん開発用ウェブサーバーを起動してみます。

```
npm start  # or  npm run serve
```

うまくいってたらWebブラウザでスケルトンが表示されます。

![スクリーンショット 2016-09-07 16.15.49.png (39.5 kB)](https://img.esa.io/uploads/production/attachments/3505/2016/09/07/10856/6f8e7d31-58e8-48cf-9a0a-8f7230a22071.png)

サーバーを止めるには `Ctrl+C` します。

ちなみにリリース用にWebアプリをビルドするのは `npm run dist`。

```
npm run dist
```

`npm run serve:dist`はリリース用にビルドしたアプリをローカルサーバーでWebブラウザに表示します。

```
npm run serve:dist
```

# Material UIを導入する

## インストール

[material-ui](https://github.com/callemall/material-ui)をインストールします。

```
npm install material-ui  --save
npm install react-tap-event-plugin --save
```

## Headerコンポーネントを作成する

```
yo react-webpack:component Header
```

`my-project/src/components/HeaderComponent.js`を以下のように編集します。

```javascript
'use strict';

import React from 'react';
import AppBar from 'material-ui/AppBar';

require('styles//Header.css');

class HeaderComponent extends React.Component {
  render() {
    return (
      <AppBar
          title="My Project"
          iconClassNameRight="muidocs-icon-navigation-expand-more"
      />
    );
  }
}

HeaderComponent.displayName = 'HeaderComponent';

// Uncomment properties you need
// HeaderComponent.propTypes = {};
// HeaderComponent.defaultProps = {};

export default HeaderComponent;
```

## メインページにヘッダを追加する

`my-project/src/components/Main.js` を以下のように編集します。

```javascript
require('normalize.css/normalize.css');
require('styles/App.css');

import React from 'react';
import MuiThemeProvider from 'material-ui/styles/MuiThemeProvider';
import RaisedButton from 'material-ui/RaisedButton';
import injectTapEventPlugin from 'react-tap-event-plugin';
import Header from './HeaderComponent';

injectTapEventPlugin();

let yeomanImage = require('../images/yeoman.png');

class AppComponent extends React.Component {
  render() {
    return (
      <MuiThemeProvider>
        <div>
          <Header />
          <div className="index">
            <img src={yeomanImage} alt="Yeoman Generator" />
            <div style={{textAlign:'center'}}>
              <RaisedButton label="OK" primary={true} />
            </div>
          </div>
        </div>
      </MuiThemeProvider>
    );
  }
}

AppComponent.defaultProps = {
};

export default AppComponent;
```

`my-project/src/styles/App.css` を以下のように編集します

```css
/* Base Application Styles */
body {
  color: #666;
  background: #fff;
}

.index img {
  margin: 40px auto;
  border-radius: 4px;
  background: #fff;
  display: block;
}

.index .notice {
  margin: 20px auto;
  padding: 15px 0;
  text-align: center;
  border: 1px solid #000;
  border-width: 1px 0;
  background: #666;
}
```

すべて保存するとこうなる

![スクリーンショット 2016-04-16 22.50.37.png (31.8 kB)](https://img.esa.io/uploads/production/attachments/3505/2016/04/16/10856/5b9efbc4-d69a-4ad5-84a6-51dd9f9b3f6c.png)

マテリアルデザインなヘッダが追加されました。

# まとめ

React+Material UIでWebアプリを開発するためのテンプレと開発ワークフローを一瞬で整えることができました。  
Android向けのReactアプリをさくっと開発するには良さそうです。

generator-react-webpackの開発用Webサーバーはファイルの変更を監視して変更を検知するとWebブラウザをオートリロードしてくれます。これはgenerator-webappなどと同じなんですが、generator-react-webpackの場合はたまに更新がうまく反映されないことがあります。  
なんかおかしいなと思ったら手動リロードしてみるのが無難ですね。個人的にはあまりストレスはなかったですが。
