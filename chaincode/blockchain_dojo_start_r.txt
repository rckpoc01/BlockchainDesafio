/*
Copyright IBM Corp 2016 All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

		 http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

/*
Implementação iniciada por Caue Garcia Polimanti e Vitor Diego dos Santos de Sousa
*/

// nome do package
package main

// lista de imports
// "encoding/json"	-> enconding para json
// "strconv" -> conversor de strings
import (
	"encoding/json"
	"errors"
	"fmt"
	"strconv"
	
	"github.com/hyperledger/fabric/core/chaincode/shim"	
)

// BoletoPropostaChaincode - implementacao do chaincode
type BoletoPropostaChaincode struct {
}



// Definição da Struct Proposta e parametros para exportação para JSON
type Proposta struct {
	IdProposta                string  `json:"idProposta"`
	DadosAceite               string  `json:"dadosAceite"`
	CpfPagador                string  `json:"cpfPagador"`
	CNPJBeneficiario		  string  `json:"cnpjBeneficiario"`	
	BoletoEmitido             bool    `json:"boletoEmitido"`
	AssinaturaBeneficiario    bool    `json:"assinaturaBeneficiario"`
	AssinaturaIFBeneficiario  bool    `json:"assinaturaIFBeneficiario"`
}

// consts associadas à tabela de Propostas
const (
	tableName                   = "Proposta"
	colIdProposta               = "idProposta"
	colDadosAceite              = "dadosAceite"
	colCpfPagador               = "cpfPagador"
	colCNPJBeneficiario         = "cnpjBeneficiario"
	colBoletoEmitido            = "boletoEmitido"
	colAssinaturaBeneficiario   = "assinaturaBeneficiario"
	colAssinaturaIFBeneficiario = "assinaturaIFBeneficiario"
)

// ============================================================================================================================
// Main
// ============================================================================================================================
func main() {
	err := shim.Start(new(BoletoPropostaChaincode))
	if err != nil {
		fmt.Printf("Error starting BoletoPropostaChaincode chaincode: %s", err)
	}
}

// ============================================================================================================================
// Init
// 		Inicia/Reinicia a tabela de propostas
// ============================================================================================================================
func (t *BoletoPropostaChaincode) Init(stub shim.ChaincodeStubInterface, function string, args []string) ([]byte, error) {
	fmt.Println("Init Chaincode...")

	// Verificação da quantidade de argumentos recebidos
	if len(args) != 0 {
		return nil, errors.New("Número de argumentos incorreto.")
	}

	// Verifica se a tabela 'Proposta' existe
	fmt.Println("Verificando se a tabela 'Proposta' existe...")
	table, err := stub.GetTable(tableName)
	
	// Se a tabela 'Proposta' já existir, excluir a tabela
	if table != nil {
		err := stub.DeleteTable(tableName)
		if err != nil {
			return nil, fmt.Errorf("Erro ao excluir tabela %s", err)
		}
	}
	
	// Criar tabela de Propostas
	fmt.Println("Criando a tabela 'Proposta'...")

	var columnDefs []*shim.ColumnDefinition
	colIdPropostaDef               := shim.ColumnDefinition{Name: colIdProposta, Type: shim.ColumnDefinition_STRING, Key: true}
	colDadosAceiteDef              := shim.ColumnDefinition{Name: colDadosAceite, Type: shim.ColumnDefinition_STRING, Key: false}
	colCpfPagadorDef               := shim.ColumnDefinition{Name: colCpfPagador, Type: shim.ColumnDefinition_STRING, Key: true}
	colCNPJBeneficiario            := shim.ColumnDefinition{Name: colCNPJBeneficiario, Type: shim.ColumnDefinition_STRING, Key: true}
	colBoletoEmitidoDef            := shim.ColumnDefinition{Name: colBoletoEmitido, Type: shim.ColumnDefinition_BOOL, Key: false}
	colAssinaturaBeneficiarioDef   := shim.ColumnDefinition{Name: colAssinaturaBeneficiario, Type: shim.ColumnDefinition_BOOL, Key: false}
	colAssinaturaIFBeneficiarioDef := shim.ColumnDefinition{Name: colAssinaturaIFBeneficiario, Type: shim.ColumnDefinition_BOOL, Key: false}

	columnDefs = append(columnDefs, &colIdPropostaDef)
	columnDefs = append(columnDefs, &colDadosAceiteDef)
	columnDefs = append(columnDefs, &colCpfPagadorDef)
	columnDefs = append(columnDefs, &colCNPJBeneficiario)
	columnDefs = append(columnDefs, &colBoletoEmitidoDef)
	columnDefs = append(columnDefs, &colAssinaturaBeneficiarioDef)
	columnDefs = append(columnDefs, &colAssinaturaIFBeneficiarioDef)
	
	createErr := stub.CreateTable(tableName, columnDefs)
	if createErr != nil {
		return nil, fmt.Errorf("Erro ao criar tabela %s", err)
	}

	fmt.Println("Tabela 'Proposta' criada com sucesso.")

	fmt.Println("Init Chaincode... Finalizado!")

	return nil, nil
}


// ============================================================================================================================
// Invoke Functions
// ============================================================================================================================

// Invoke - Ponto de entrada para chamadas do tipo Invoke.
// Funções suportadas:
// "init": inicializa o estado do chaincode, também utilizado como reset
// "registrarProposta(Id, cpfPagador, pagadorAceitou, beneficiarioAceitou, 
// boletoPago)": para registrar uma nova proposta ou atualizar uma já existente.
func (t *BoletoPropostaChaincode) Invoke(stub shim.ChaincodeStubInterface, function string, args []string) ([]byte, error) {
	fmt.Println("Invoke Chaincode...")
	fmt.Println("invoke is running " + function)

	// Estrutura de Seleção para escolher qual função será executada, 
	// de acordo com a funcao chamada
	switch function {
		case "init":
		return t.Init(stub, "init", args) 

		case "registrarProposta":
		return t.registrarProposta(stub, args)
	}

	fmt.Println("invoke não encontrou a func: " + function) //error

	return nil, errors.New("Invocação de função desconhecida: " + function)
}

// registrarProposta: função Invoke para registrar uma nova proposta, recebendo os seguintes argumentos:
// args[0]: Id. Hash que identificará a proposta
// args[1]: cpfPagador. CPF do Pagador
// args[2]: pagadorAceitou. Status de aceite do Pagador da proposta
// args[3]: beneficiarioAceitou. Status de aceite do Beneficiario da proposta
// args[4]: boletoPago. Status do Pagamento do Boleto
func (t *BoletoPropostaChaincode) registrarProposta(stub shim.ChaincodeStubInterface, args []string) ([]byte, error) {
	fmt.Println("registrarProposta...")

	// Verifica se a quantidade de argumentos recebidas corresponde a esperada
	if len(args) != 7 {
		return nil, errors.New("Incorrect len.")
	}

	// Obtem os valores da array de arguments (args) e 
	// os converte no tipo necessário para salvar na tabela 'Proposta'
	idProposta       := args[0]
	dadosAceite      := args[1]
	cpfPagador       := args[2]
	cnpjBeneficiario := args[3]

	boletoEmitido, err := strconv.ParseBool(args[4])
	if err != nil {
		return nil, errors.New("registrarProposta failed. ParseBool failed")
	}
	assinaturaBeneficiario, err := strconv.ParseBool(args[5])
	if err != nil {
		return nil, errors.New("registrarProposta failed. ParseBool failed")
	}
	assinaturaIFBeneficiario, err := strconv.ParseBool(args[6])
	if err != nil {
		return nil, errors.New("registrarProposta failed. ParseBool failed")
	}

	fmt.Printf("registrarProposta(%s, %s, %s, %s, %s, %s, %s)", args[0], args[1], args[2], args[3], args[4], args[5], args[6])


	// Registra a proposta na tabela 'Proposta'
	var columns []*shim.Column
	col1 := shim.Column{Value: &shim.Column_String_{String_: idProposta}}
	col2 := shim.Column{Value: &shim.Column_String_{String_: dadosAceite}}
	col3 := shim.Column{Value: &shim.Column_String_{String_: cpfPagador}}
	col4 := shim.Column{Value: &shim.Column_String_{String_: cnpjBeneficiario}}
	col5 := shim.Column{Value: &shim.Column_Bool{Bool: boletoEmitido}}
	col6 := shim.Column{Value: &shim.Column_Bool{Bool: assinaturaBeneficiario}}
	col7 := shim.Column{Value: &shim.Column_Bool{Bool: assinaturaIFBeneficiario}}

	columns = append(columns, &col1)
	columns = append(columns, &col2)
	columns = append(columns, &col3)
	columns = append(columns, &col4)
	columns = append(columns, &col5)
	columns = append(columns, &col6)
	columns = append(columns, &col7)

	row := shim.Row{Columns: columns}
	ok, err := stub.InsertRow(tableName, row)
	if err != nil {
		return nil, fmt.Errorf("registrarProposta (insert) failed. %s", err)
	}
	
	// Caso a proposta já exista
	if !ok {
		// Trecho para atualizar uma proposta existente
		// substitui um registro existente em uma linha com o registro associado ao idProposta recebido nos argumentos
		ok, err := stub.ReplaceRow(tableName, row)
		if err != nil {
			return nil, fmt.Errorf("registrarProposta (replace) failed. %s", err)
		}
		if !ok {
			return nil, errors.New("registrarProposta (replace) failed. Row with given key does not exist")
		}
	}
	
	fmt.Println("Proposta criada!")

	return nil, nil
}


// ============================================================================================================================
// Query
// ============================================================================================================================

// Query is our entry point for queries

// Query - Ponto de entrada para chamadas do tipo Query.
// Funções suportadas:
// "consultarProposta(Id)": para consultar uma proposta existente
func (t *BoletoPropostaChaincode) Query(stub shim.ChaincodeStubInterface, function string, args []string) ([]byte, error) {
	fmt.Println("Query Chaincode...")

	fmt.Println("query is running " + function)

	// Estrutura de Seleção para escolher qual função será executada, 
	// de acordo com a funcao chamada

	switch function {

		case "consultarProposta":
		return t.consultarProposta(stub, args) 

	}

	fmt.Println("query encontrou a func: " + function) 

	return nil, errors.New("Query de função desconhecida: " + function)
}

// consultarProposta: função Query para consultar uma proposta existente, recebendo os seguintes argumentos
// args[0]: Id. Hash da proposta
func (t *BoletoPropostaChaincode) consultarProposta(stub shim.ChaincodeStubInterface, args []string) ([]byte, error) {
	fmt.Println("consultarProposta...")
	
	//var resProposta Proposta		// Proposta
	var propostaAsBytes []byte		// retorno do json em bytes


	// Verifica se a quantidade de argumentos recebidas corresponde a esperada
	if len(args) < 1 {
		return nil, errors.New("consultarProposta failed. Must include 1 key value")
	}

	// Obtem os valores dos argumentos e os prepara para salvar na tabela 'Proposta'
	idProposta  := args[0]

	// Define o valor de coluna do registro a ser buscado
	var columns []shim.Column
	col1 := shim.Column{Value: &shim.Column_String_{String_: idProposta}}
	columns = append(columns, col1)

	// Consultar a proposta na tabela 'Proposta'
	rowChannel, err := stub.GetRows(tableName, columns)
			

	// Tratamento para o caso de não encontrar nenhuma proposta correspondente
	if err != nil {
		return nil, fmt.Errorf("consultarProposta operation failed. %s", err)
	}

	var rows []shim.Row
	for {
		select {
		case row, ok := <-rowChannel:
			if !ok {
				rowChannel = nil
			} else {
				rows = append(rows, row)
			}
		}
		if rowChannel == nil {
			break
		}
	}

	var propostas []Proposta

	for i := 0; i < len(rows); i++ {
		dadosAceite               := rows[i].Columns[1].GetString_()
		cpfPagador                := rows[i].Columns[2].GetString_()
		cnpjBeneficiario          := rows[i].Columns[3].GetString_()
		boletoEmitido             := rows[i].Columns[4].GetBool()
		assinaturaBeneficiario    := rows[i].Columns[5].GetBool()
		assinaturaIFBeneficiario  := rows[i].Columns[6].GetBool()

		// Converter o objeto da Proposta para Bytes, para retorná-lo em formato JSON

		proposta := Proposta{IdProposta               : idProposta, 
							DadosAceite               : dadosAceite,
							CpfPagador                : cpfPagador,
							CNPJBeneficiario          : cnpjBeneficiario,
							BoletoEmitido             : boletoEmitido,
							AssinaturaBeneficiario    : assinaturaBeneficiario,
							AssinaturaIFBeneficiario  : assinaturaIFBeneficiario}

		propostas = append(propostas, proposta)
	}
	

	// retorna o objeto em bytes
	
	propostaAsBytes, err = json.Marshal(propostas)
	if err != nil {
		return nil, fmt.Errorf("consultarProposta operation failed. Error marshaling JSON: %s", err)
	}

	return propostaAsBytes, nil
	//return propostaAsBytes, nil
}

