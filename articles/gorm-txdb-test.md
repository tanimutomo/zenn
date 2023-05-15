---
title: "Gormのテストで，シード挿入と並列実行をいい感じにやる"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "test", "Gorm", "並列"]
published: true
publication_name: "ispec_inc"
---

# Gormのテストで，シード挿入と並列実行をいい感じにやりたい

内容は，ざっくり以下2つです
- Daoパッケージのテストを並列実行する
- テストの冒頭でいい感じにシードを挿入する

サンプルコードです．

https://github.com/tanimutomo/gorm-txdb


## Daoパッケージのテストを並列実行する

DaoなどのDBに絡むパッケージのテストでは，データベースという共有資源を利用する必要があるため，並列に実行することは通常できないですよね．
ここは，[DATA-DOG/go-txdbでDB接続を含むテストを楽に書こう](https://zenn.dev/rinchsan/articles/83cf6f3b5d70c4d9b5d4) を参考にして，[DATA-DOG/go-txdb](https://github.com/DATA-DOG/go-txdb) を利用することで，結構いい感じに解決できました．
この記事では，txdbの詳細な解説はしません．
txdbについては，レポジトリか，上の記事をご覧ください．
本記事では，txdbをgormで利用する場合の方法を重点的に書きます．


## テストの冒頭でいい感じにシードを挿入する

テストごとにいい感じにテストデータを挿入していきたいわけですが，これにはいくつか選択肢があると思います．
私は初めは，[go-testfixtures/testfixtures](https://github.com/go-testfixtures/testfixtures) を利用していましたが，yamlファイルが増えすぎるのが嫌なのと，テーブルの中身をdeleteする処理が面倒だったので，sqlのファイルを直接実行できる [tanimutomo/sqlfile](https://github.com/tanimutomo/sqlfile) を作ってこれを利用していました．
しかし，やっぱりSQLは人間にとって書きやすいものではないですよね，，，

そこで，色々考えていたんですが，`*gorm.DB.Create` メソッドの引数が，`interface{}`であることに気づき，`[]interface{}{...}` としてシードを定義すればいい感じにテストデータを定義できて挿入処理もできるのではと思い，やってみることにしました．


# テストコード

とりあえず，実際のdaoのテストコードをみてもらったほうが早いと思うので乗せます．

https://github.com/tanimutomo/gorm-txdb/blob/main/pkg/dao/user_test.go

GetとCreateのメソッドと，そのサブテストを全て，`t.Parallel()` で並列化してます．

```go
// pkg/dao/user_test.go

package dao

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/tanimutomo/gorm-txdb/pkg/entity"
)

func TestUserDao_Get(t *testing.T) {
	t.Parallel()

	seeds := []interface{}{
		&entity.User{ID: int64(1), Name: "foo"},
	}
	db, err := prepare("user_dao_get", seeds)
	if err != nil {
		t.Fatal(err)
	}
	d := NewUser(db)

	tests := []struct {
		name   string
		give   int64
		wantID int64
		err    bool
	}{
		{
			name:   "success",
			give:   int64(1),
			wantID: int64(1),
		},
		{
			name: "not found",
			give: int64(2),
			err:  true,
		},
	}
	for _, tt := range tests {
		tt := tt
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()

			got, aerr := d.Get(tt.give)
			if tt.err {
				assert.Error(t, aerr)
			} else {
				assert.NoError(t, aerr)
				assert.Equal(t, tt.wantID, got.ID)
			}
		})
	}
}

func TestUserDao_Create(t *testing.T) {
	t.Parallel()

	seeds := []interface{}{
		&entity.User{Name: "exist"},
	}
	db, err := prepare("user_dao_create", seeds)
	if err != nil {
		t.Fatal(err)
	}
	d := NewUser(db)

	tests := []struct {
		name string
		give string
		err  bool
	}{
		{
			name: "success",
			give: "non exist",
		},
		{
			name: "duplicated",
			give: "exist",
			err:  true,
		},
	}
	for _, tt := range tests {
		tt := tt
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()

			aerr := d.Create(tt.give)
			if tt.err {
				assert.Error(t, aerr)
			} else {
				assert.NoError(t, aerr)
				assert.NoError(t, db.First(&entity.User{}, "name = ?", tt.give).Error)
			}
		})
	}
}
```


今回の肝は，各テスト関数のここです．

```go
	seeds := []interface{}{
		&entity.User{ID: int64(1), Name: "foo"},
	}
	db, err := prepare("user_dao_get", seeds)
	if err != nil {
		t.Fatal(err)
	}
	d := NewUser(db)
```


# 説明

ここから上のコードについてと，gormでのtxdbの設定について，軽く説明をしていきます．

## txdbで登録したDBを呼び出す

テスト関数では，`*gorm.DB` を取得するために，`prepare(テストデータベース名(テストごとに個別の名前にする), seeds)` 関数を呼んでいます．

```go
	db, err := prepare("user_dao_get", seeds)
	if err != nil {
		t.Fatal(err)
	}
```


`prepare(...)` 関数はこんな感じです．

```go
package dao

import (
	"github.com/tanimutomo/gorm-txdb/pkg/mysql"
	"gorm.io/gorm"
)

func prepare(name string, seeds []interface{}) (*gorm.DB, error) {
	db, err := mysql.NewTest(name)
	if err != nil {
		return nil, err
	}
	for _, s := range seeds {
		if err := db.Create(s).Error; err != nil {
			return nil, err
		}
	}
	return db, nil
}
```

prepare関数が読んでいる `mysql.NewTest` はこんな感じです．


```go
package mysql

import (
	"fmt"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"

	"github.com/DATA-DOG/go-txdb"
	"github.com/tanimutomo/gorm-txdb/pkg/config"
)

func NewTest(name string) (*gorm.DB, error) {
	dsn := fmt.Sprintf(
		"%s:%s@(%s:%s)/%s?charset=utf8mb4&parseTime=true",
		config.RDS.User, config.RDS.Password,
		config.RDS.Host, config.RDS.Port,
		config.RDS.Database,
	)

	txdb.Register(name, "mysql", dsn)
	dialector := mysql.New(mysql.Config{
		DriverName: name,
		DSN:        dsn,
	})

	return gorm.Open(dialector, &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
}
```

おそらく，通常のGormのDBの初期化と違う点は，以下の行です．
ここでは，txdbで登録したトラザクションをドライバとしてmysqlに登録しています（理解が間違っていたら教えてください）．
ここの `name` 変数に入っているのは，各テストで個別に定義された名前なので，ここのテストが別のトランザクション上で走るため，あたかも別のDBを扱っているような体験になり，更新系の処理を含むテストがあっても，関係なく並列化することができます．

```go
	txdb.Register(name, "mysql", dsn)
	dialector := mysql.New(mysql.Config{
		DriverName: name,
		DSN:        dsn,
	})
```


## テストごとにいい感じにテストデータを挿入する

テスト関数では，`*gorm.DB` を取得するために，`prepare(testDBName, seeds)` を呼んでいます．
seedsは，`[]interface{}{ Gormのモデルs }` として定義します．
seedsに，事前に挿入しておきたいデータ全て入れます．

```go
	seeds := []interface{}{
		&entity.User{ID: int64(1), Name: "foo"},
	}
	db, err := prepare("user_dao_get", seeds)
	if err != nil {
		t.Fatal(err)
	}
```


先ほど示した通り，`prepare(...)` 関数はこんな感じで，`seeds`をイテレートして，`*gorm.DB.Create`で挿入していきます．
ここで挿入されたデータは，txdbで定義したDBに書き込んでいくため，
- 他のテストには影響を与えず，
- テストが終わったら自動的に削除され
ます．


```go
package dao

import (
	"github.com/tanimutomo/gorm-txdb/pkg/mysql"
	"gorm.io/gorm"
)

func prepare(name string, seeds []interface{}) (*gorm.DB, error) {
	db, err := mysql.NewTest(name)
	if err != nil {
		return nil, err
	}
	for _, s := range seeds {
		if err := db.Create(s).Error; err != nil {
			return nil, err
		}
	}
	return db, nil
}
```


# まとめ

Gormとtxdbを利用して，Daoのテストを並列化しつつ，いい感じにシードデータを入れる方法について書きました．
私も試行錯誤中なので，もっと他にいい方法を知っている方がいれば，教えていただけると嬉しいです．
また，間違っているところがあれば，ご指摘いただければと思います．

いいテストライフを過ごしましょう！📑
