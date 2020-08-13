# oasSample

OpenAPI-Generator よる Go 言語用サービスの構築テスト

## 目的

OpenAPI-Generator で、python だと API 修正などをして再生成すると、
コードの上書きを気を付けるのが面倒なので、Go 言語を使って実装してみる.

## OpenAPI-Generator 側の作業

### 条件

-   使用 OpenAPI-Generator バージョン
    -   openapi-generator-5.0.0-beta
-   GO 言語 1.13 以降 (go.mod 必須)

### template の修正

1. OpenAPI-Generator のソースコードの取得

    https://github.com/OpenAPITools/openapi-generator/releases
    から、対応するバージョンのソースコードを取得する

2. コードテンプレートの抽出

    openapi-generator/modules/openapi-generator/src/main/resources/
    から、取得した Generator のテンプレートを取得
    今回の場合は、go-server を持ってくる

3. テンプレートの修正

    以下のようにテンプレートを修正する。

    - go.mod.mustache
        - モジュール名が、github を参照してしまうので、gitRepoId だけに変更.
        - {{gitRepoId}}が、モジュール名をする
    - main.mustache
        - 生成したコードのモジュール名を gitRepoId に統一

    ```Diff
    --- go.mod.mustache.orig
    +++ go.mod.mustache
    @@ -1,4 +1,4 @@
    -module {{gitHost}}/{{gitUserId}}/{{gitRepoId}}
    +module {{gitRepoId}}

    go 1.13

    --- main.mustache.orig
    +++ main.mustache
    @@ -5,7 +5,7 @@
        "log"
        "net/http"

    -	{{packageName}} "{{gitHost}}/{{gitUserId}}/{{gitRepoId}}/{{sourceFolder}}"
    +	{{packageName}} "{{gitRepoId}}/{{sourceFolder}}"
    )

    func main() {
    ```

### OpenAPI-Generator 設定ファイル

1. OpenAPI-Generator 用の設定項目の確認
   java -jar ./openapi-generator-cli.jar config-help -g {{generator-name}}
   で、各 generator 用の設定ファイル項目が出力される。

    go-server の場合

    ```text
    $ java -jar ./openapi-generator-cli.jar config-help -g go-server

    CONFIG OPTIONS

       featureCORS
           Enable Cross-Origin Resource Sharing middleware (Default: false)

       hideGenerationTimestamp
           Hides the generation timestamp when files are generated. (Default: true)

       packageName
           Go package name (convention: lowercase). (Default: openapi)

       packageVersion
           Go package version. (Default: 1.0.0)

       serverPort
           The network port the generated server binds to (Default: 8080)

       sourceFolder
           source folder for generated code (Default: go)
    ```

2. OpenAPI-Generator 設定ファイルの作成

    1 の結果を元に、yaml を作成する.
    go-server の場合、以下のような形(go-server-config.yaml)

    ```yaml
    # Enable Cross-Origin Resource Sharing middleware (Default: false)
    featureCORS: true

    # Hides the generation timestamp when files are generated. (Default: true)
    hideGenerationTimestamp: true

    # Go package name (convention: lowercase). (Default: openapi)
    packageName: openapi

    # Go package version. (Default: 1.0.0)
    packageVersion: 1.0.0

    # The network port the generated server binds to (Default: 8080)
    serverPort: 3000

    # source folder for generated code (Default: go)
    sourceFolder: openapi
    ```

### OpenAPI-Generator の起動

1. OpenAPI-Generator の起動

    テンプレート(-t)と設定ファイルを指定(-c) して生成する

    ```sh
    $ java -jar ./openapi-generator-cli.jar generate -g go-server -i ./schema/reference/oasSample.v1.yaml -t ./schema/template/go-server --git-repo-id oasSample -o ./oasSample -c ./go-server-config.yaml
    ```

## サーバ側のコード

### 出力コード

先の生成コマンドを実行すると以下のようなツリー構造が出力される。

```
oasSample
  |-  .openapi-generator-ignore
  |-  Dockerfile
  |-  go.mod
  |-  go.sum
  |-  main.go
  |-  README.md
  |
  +---.openapi-generator
  |   |-  FILES
  |   \-  VERSION
  |
  +---api
  |   \-  openapi.yaml
  |
  \---openapi
      |-  api.go
      |-  api_sampledata.go
      |-  api_sampledata_service.go
      |-  logger.go
      |-  model_sample_data.go
      |-  model_sample_response.go
      \-  routers.go
```

### 実装コードの追加

1.  実装モジュール用フォルダの作成

    生成コードと干渉しないように別フォルダでモジュールを作成する。

    ```sh
    # 例)実装モジュール用にappフォルダを作成.
    $ cd src
    $ mkdir app
    ```

2.  app フォルダに実装コードをコピー

    実装コードは、 api\_{{tags}}\_service.go をベースとする。
    {{tags}}は、OpenAPI 仕様上の各 API に付ける tags に由来する。

    OpenAPI 仕様上で、定義がない場合は、 {{tags}}= default 　となるため、
    api_default_service.go となる。

    ```sh
    # 元の名前と区別するため、api_を外してコピーする
    $ cp openapi/api_sampledata_service.go app/sampledata_service.go
    ```

    > 実際には、 openapi/api.go で定義されている 各 ApiServicer の interface を実装するクラスを作成するのが正しい.

3.  実装コードの修正

    実装コードのモジュール名は、フォルダ名に合わせ app とする.

    -   修正箇所(上から順に)
        1. パッケージ名の変更
        2. 自動生成コードの import
        3. interface クラスの参照パッケージ名変更
        4. 引数の参照パッケージ名変更

    ```diff
    --- oasSample/openapi/api_sampledata_service.go
    +++ oasSample/app/sampledata_service.go
    @@ -9,3 +9,3 @@

    -package openapi
    +package app

    @@ -13,2 +13,3 @@
        "errors"
    +	openapi "oasSample/openapi"
    )
    @@ -22,3 +23,3 @@
    // NewSampledataApiService creates a default api service
    -func NewSampledataApiService() SampledataApiServicer {
    +func NewSampledataApiService() openapi.SampledataApiServicer {
        return &SampledataApiService{}
    @@ -34,3 +35,3 @@
    // PutSampleDataDataId - Update SampleData
    -func (s *SampledataApiService) PutSampleDataDataId(dataId string, sampleData SampleData) (interface{}, error) {
    +func (s *SampledataApiService) PutSampleDataDataId(dataId string, sampleData openapi.SampleData) (interface{}, error) {
        // TODO - update PutSampleDataDataId with the required logic for this service method.

    ```

4.  main.go を修正

    main 関数で、実装コードを上書きするように設定変更する

    ```Diff
    --- oasSample/main.go.orig
    +++ oasSample/main.go
    @@ -13,14 +13,13 @@
        "log"
        "net/http"

    +	app "oasSample/app"
        openapi "oasSample/openapi"
    )

    func main() {
        log.Printf("Server started")

    -	SampledataApiService := openapi.NewSampledataApiService()
    +	SampledataApiService := app.NewSampledataApiService()
        SampledataApiController := openapi.NewSampledataApiController(SampledataApiService)

        router := openapi.NewRouter(SampledataApiController)

    ```

    やっていることは、
    ApiService の生成関数を、openapi(自動生成)から、app 　に変更しているだけ

    当然、色々命名変更しても問題ない。要するに 先の interface と同等のメソッドを持つ構造体のインスタンスを
    openapi.NewDataComparisonApiController に渡すようにするだけ

    また、複数の API タグがある場合は、それぞれ作成する

5.  .openapi-generator-ignore 　の修正
    .openapi-generator-ignore の最終行に　 main.go 　を追加する

---

ここまでで、再度生成コマンドを実行しても、main.go も実装コマンドも上書きされることはない

### swagger-ui の追加
1. html/ui ディレクトリの追加
2. html/uiにswagger-uiのファイルを追加
3. main.go に　html/ui への 静的ファイル参照を追加
4. main.go に　api/openapi.yaml (OpenAPI-Generatorが自動生成したyaml)　への静的ファイル参照を追加i
5. html/ui/index.html のjavascript,css, openapi.yaml への参照を修正
