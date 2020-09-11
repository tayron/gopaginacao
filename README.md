# Go Paginação
Biblioteca para gerar paginação


![Alt text](/print.png?raw=true)

## Importação
Para importar basta execuar o comando: ```go get -u github.com/tayron/gopaginacao```

## Configuração
Deve-se criar variavel de ambiente com o valor ```NUMERO_REGISTRO_POR_PAGINA=5``` portanto recomenda-se criação de um arquivo ```.env``` 
para este tipo de configuração.

## Alterando template
Caso deseja-se alterar o template, deve-se realizar um fork deste projeto e alterar o template no arquivo **gopaginacao.go**. 
No inicio do arquivo segue as constantes com layout que seguem modelo de paginação do Twitter Bootstrap 4 (https://getbootstrap.com/docs/4.0/components/pagination)

```
const estruturaContainerMenu = "<nav><ul class='pagination'>%s</ul></nav>Você está na página %d, exibindo %d de %d registros"
const estruturaItemMenu = "<li class='page-item'><a class='page-link' href='%s'>%s</a></li>"
const estruturaItemMenuSelecionado = "<li class='page-item active'><a class='page-link' href='%s'>%s</a></li>"
const estruturaItemMenuDesabilitado = "<li class='page-item disabled'><a class='page-link' href='%s'>%s</a></li>"
```

## Como utilizar
No handler basta informar o **número total de registros** que existe no banco de dados, também deve ser necessário informar 
o **objeto de requisição: http.Request**  que será necessário para recuperar qual página foi selecionada.

Exemplo: 

```
numeroTotalRegistro := models.ObterNumeroProdutos()
htmlPaginacao, offset, err := gopaginacao.CriarPaginacao(numeroTotalRegistro, r)
```

A função ```CriarPaginacao(numeroTotalRegistro, r)``` irá retornar três informações:
* O Html da páginação que deverá ser impresso na tela, ela contém o html da páginação e deve ser informado como parametro 
usando a função **emplate.HTML()** do pacote **"html/template"**, exemplo: 

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
	numeroTotalRegistro := models.ObterNumeroProdutos()
	htmlPaginacao, offset, err := gopaginacao.CriarPaginacao(numeroTotalRegistro, r)

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
