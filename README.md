# Projeto: Construção de Star Schema para Cenários Universitários
### Bootcamp NTT DATA - Engenharia de Dados com Python (DIO)

Este repositório documenta a transformação de um modelo de banco de dados relacional (transacional) em um modelo dimensional (**Star Schema**), otimizado para análise de indicadores de professores e disciplinas em um ambiente universitário.

---

## 1. Objetivo do Desafio

O desafio consistiu em:

* Ignorar dados operacionais irrelevantes para este contexto (como dados de alunos e pré-requisitos).
* Estruturar uma Tabela Fato que consolide as métricas de ensino.
* Criar Dimensões que permitam filtrar essas métricas por Curso, Disciplina, Professor, Departamento e Tempo.

---

## 🛠️ 2. Infraestrutura: Banco de Dados Relacional (MySQL)

Para dar suporte ao projeto, os dados foram inicialmente estruturados e populados em um ambiente MySQL. Abaixo estão os scripts utilizados:

### 📜 Script 01: Criação das Tabelas (DDL)
```sql
CREATE DATABASE IF NOT EXISTS universidade;
USE universidade;

-- Tabela Departamento
CREATE TABLE Departamento (
    idDepartamento INT PRIMARY KEY,
    Nome VARCHAR(45),
    Campus VARCHAR(45)
);

-- Tabela Professor
CREATE TABLE Professor (
    idProfessor INT PRIMARY KEY,
    Nome VARCHAR(45),
    Departamento_idDepartamento INT,
    FOREIGN KEY (Departamento_idDepartamento) REFERENCES Departamento(idDepartamento)
);

-- Tabela Curso
CREATE TABLE Curso (
    idCurso INT PRIMARY KEY,
    Nome VARCHAR(45)
);

-- Tabela Disciplina
CREATE TABLE Disciplina (
    idDisciplina INT PRIMARY KEY,
    Nome VARCHAR(45),
    Professor_idProfessor INT,
    Curso_idCurso INT,
    FOREIGN KEY (Professor_idProfessor) REFERENCES Professor(idProfessor),
    FOREIGN KEY (Curso_idCurso) REFERENCES Curso(idCurso)
);

-- Tabela Associativa Disciplina_Curso (Base para a Fato)
CREATE TABLE Disciplina_Curso (
    Disciplina_idDisciplina INT,
    Curso_idCurso INT,
    PRIMARY KEY (Disciplina_idDisciplina, Curso_idCurso),
    FOREIGN KEY (Disciplina_idDisciplina) REFERENCES Disciplina(idDisciplina),
    FOREIGN KEY (Curso_idCurso) REFERENCES Curso(idCurso)
);
```

### 📜 Script 02: Inserção de Dados (DML)
```sql
-- Populando Departamentos
INSERT INTO Departamento VALUES (1, 'Ciência da Computação', 'Campus A'), (2, 'Engenharia Elétrica', 'Campus B');

-- Populando Professores
INSERT INTO Professor VALUES (10, 'Arthur Haerdy Jr', 1), (20, 'Diógenes Pardo', 1), (30, 'José Augusto', 2);

-- Populando Cursos
INSERT INTO Curso VALUES (100, 'Bacharelado em TI'), (200, 'Engenharia de Software');

-- Populando Disciplinas
INSERT INTO Disciplina VALUES (500, 'Banco de Dados', 10, 100), (501, 'Python para Dados', 10, 200), (502, 'Circuitos Lógicos', 30, 100);

-- Populando a Tabela Associativa
INSERT INTO Disciplina_Curso VALUES (500, 100), (501, 200), (502, 100);
```

---

## ⚙️ 3. Processo de ETL (Power Query)

Após a importação dos dados para o Power BI, as seguintes transformações foram aplicadas no **Editor do Power Query**:

1.  **Limpeza e Renomeação:** Tabelas originais foram renomeadas para o prefixo `Dim_` (ex: `Dim_Professor`). Colunas desnecessárias foram removidas para otimizar a performance.
2.  **Criação da Dim_Data:** Como o banco original não possuía datas, foi criada uma dimensão temporal via Script M para permitir análises por ano, mês e trimestre:

    ```powerquery
    let
        DataInicio = #date(2025, 1, 1),
        DataFim = #date(2025, 12, 31),
        ListaDatas = List.Dates(DataInicio, Duration.Days(DataFim - DataInicio) + 1, #duration(1,0,0,0)),
        Tabela = Table.FromList(ListaDatas, Splitter.SplitByNothing(), {"Data"}),
        TipoData = Table.TransformColumnTypes(Tabela, {{"Data", type date}}),
        DataKey = Table.AddColumn(TipoData, "DataKey", each Date.Year([Data])*10000 + Date.Month([Data])*100 + Date.Day([Data]), Int64.Type)
    in
        DataKey
    ```
3.  **Construção da Fato_Professor:**
    * Foi utilizada a tabela `Disciplina_Curso` como base (Referência).
    * Foi aplicado o comando **Mesclar Consultas** para trazer o `Professor_idProfessor` da tabela de Disciplinas.
    * Foi adicionada uma **Coluna Personalizada** chamada `Qtd_Disciplinas` com valor fixo `1` (métrica de contagem).
    * Foi adicionada a coluna `DataKey` para relacionar com a dimensão calendário.

### Visão do Power Query Editor: Organização das consultas dimensionais e aplicação das etapas de transformação (ETL) para limpeza e criação da tabela fato:

<p align="center">
  <img src="000-Midia_e_Anexos/2026-04-15-14-37-04.png" alt="" width="1024">
</p>



---

## 📐 4. Arquitetura do Modelo (Star Schema)

O modelo final foi estruturado no formato **Snowflake/Star Schema** para garantir que o Departamento filtre o Professor e este, por sua vez, filtre a Fato.

### Relacionamentos:
* **Fato_Professor [1:N] Dim_Professor**: Via `Professor_idProfessor`.
* **Fato_Professor [1:N] Dim_Curso**: Via `Curso_idCurso`.
* **Fato_Professor [1:N] Dim_Disciplina**: Via `Disciplina_idDisciplina`.
* **Fato_Professor [1:N] Dim_Data**: Via `DataKey`.
* **Dim_Professor [1:N] Dim_Departamento**: Via `idDepartamento` (Configurando o braço Snowflake).

---

## 🚀 5. Conclusão
O modelo resultante permite uma análise granular da carga acadêmica. É possível visualizar, por exemplo, o total de disciplinas ofertadas por departamento em um período específico ou o volume de cursos atendidos por cada docente.

---
**Desenvolvido por:** Arthur Haerdy Junior  
**Contexto:** Desafio Módulo 8 - Bootcamp NTT DATA Python & Data Engineering.