# Go Paginação
Biblioteca para gerar paginação utilizando modelo bootstrap twitter 4

![Alt text](/print.png?raw=true)

## Importação
Para importar basta execuar o comando: ```go get -u github.com/tayron/go-paginacao```

## Configuração
Deve-se criar variavel de ambiente com o valor:
* NUMERO_REGISTRO_POR_PAGINA=5

## Como utilizar
No handler basta informar a URI da página, exemplo **/produtos**
e o **número total de registros** no banco de dados, também deve ser necessário informar 
o **objeto de requisição: http.Request**  que será necessário para recuperar qual página foi selecionada.

Exemplo: 

```
uri := "/produtos"
numeroTotalRegistro := models.ObterNumeroProdutos()
htmlPaginacao, offset, err := library.CriarPaginacao(uri, numeroTotalRegistro, r)
```

A função ```CriarPaginacao(uri, numeroTotalRegistro, r)``` irá retornar três informações:
* O Html da páginação que deverá ser impresso na tela, ela contém o html da páginação e deve ser informado usando a função **emplate.HTML()** do pacote **"html/template"**, exemplo: 

```
parameters := struct {
    Produtos  []models.Produto
    Paginacao template.HTML
}{
    Produtos:  listaProdutos,
    Paginacao: template.HTML(htmlPaginacao),
}  
```

* O Offset usado na consulta dos dados a serem exibidos na página selecionada, exemplo: ```models.BuscarTodosProdutos(offset)``` e a SQL deverá implementar a consulta de forma semelhante a: ```SELECT id, nome, cor, tamanho FROM produtos ORDER BY id DESC LIMIT ? OFFSET ?```, passando limite de registro configurado no arquivo .env.

* O Erro caso seja informado na páginação alguma informação inválida

## Exempĺo completo da implementação
Por fim o handler deve ser semelhante ao exemplo abaixo:

```
func ProdutoHandler(w http.ResponseWriter, r *http.Request) {
	uri := "/produtos"
	numeroTotalRegistro := models.ObterNumeroProdutos()
	htmlPaginacao, offset, err := library.CriarPaginacao(uri, numeroTotalRegistro, r)

	var listaProdutos []models.Produto

	if err == nil {
		listaProdutos = models.BuscarTodosProdutos(offset)
	}

	parameters := struct {
		Produtos  []models.Produto
		Paginacao template.HTML
	}{
		Produtos:  listaProdutos,
		Paginacao: template.HTML(htmlPaginacao),
	}  
    
    ...	resto do código ...
}      
```

Exemplo da consulta no banco:
```
func BuscarTodosProdutos(offset int) []Produto {

	db := database.ObterConexao()
	defer db.Close()

	var sql string = `SELECT id, nome, cor, tamanho FROM produtos ORDER BY id DESC LIMIT ? OFFSET ?`

	numeroRegistro := os.Getenv("NUMERO_REGISTRO_POR_PAGINA")
	rows, err := db.Query(sql, numeroRegistro, offset)

	if err != nil {
		panic(err)
	}

	defer rows.Close()

	var listaProdutos []Produto
	for rows.Next() {

		var produtoStruct Produto

		rows.Scan(&produtoStruct.ID,
			&produtoStruct.Nome,
			&produtoStruct.Cor,
			&produtoStruct.Tamanho)

		listaProdutos = append(listaProdutos, produtoStruct)
	}

	return listaProdutos
}
```

Exemplo da contagem de registros no banco:
```
func ObterNumeroProdutos() int {

	db := database.ObterConexao()
	defer db.Close()

	var sql string = `SELECT count(0) FROM produtos`

	rows, err := db.Query(sql)

	if err != nil {
		panic(err)
	}

	defer rows.Close()

	var numeroProdutos int = 0
	for rows.Next() {
		rows.Scan(&numeroProdutos)
	}

	return numeroProdutos
}
```

## Exibindo páginação no HTML
Na tela de listagem dos deve-se imprimir a páginação usando o comando ```{{.Paginacao}}```, exemplo completo:

```
<div class="row">
    <div class="col-md-12">
        <table class="table table-responsive table-bordered table-striped table-hovered">
            <thead>
                <tr>
                    <th>#</th>
                    <th>Referencia</th>
                    <th>Cor</th>
                    <th>Tamanho</th>
                </tr>
            </thead>
            <tbody>
                {{- range $produto := .Produtos -}}
                    <tr>
                        <td>{{$produto.ID}}</td>
                        <td>{{$produto.Nome}}</td>
                        <td>{{$produto.Cor}}</td>
                        <td>{{$produto.Tamanho}}</td>
                    </tr>
                {{ end }}
            </tbody>
        </table>
        {{.Paginacao}}
    </div>
</div>
```
