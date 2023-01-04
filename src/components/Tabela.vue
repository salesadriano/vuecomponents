<template>
  <div>
    <input type="hidden" :id="id + '_carregar'" @click="carregar()" />
    <input type="hidden" :id="id + '_limpar'" @click="limpar()" />
    <table
      :id="id"
      class="table table-bordered text-center datatable"
      style="width: 100%"
      :data-dataset="this.myDataset"
    >
      <thead v-if="myDataset.colunas">
        <th v-if="seq" id="seq">Ordem</th>
        <th
          v-for="coluna in myDataset.colunas"
          :key="columnId(coluna)"
          :colspan="coluna.largura"
          :rowspan="coluna.linhas"
          :onclick="coluna.funcao"
          :class="(coluna.classe || '') + ' link '"
          :id="columnId(coluna)"
          :title="retornaDescricao(coluna.descricao)"
          @click="reverseTabela(coluna.dado.origem)"
        >
          {{ coluna.nome }}
        </th>
      </thead>
      <tbody v-if="myDataset.colunas">
        <tr
          v-for="(linha, chaveRow) in myDataset.origem"
          :key="chaveRow"
          :id="rowId(chaveRow)"
        >
          <td v-if="seq">{{ chaveRow + 1 }}</td>
          <td
            v-for="(coluna, chaveCel) in myDataset.colunas"
            :key="chaveCel"
            :id="celId(chaveRow, coluna)"
            :onclick="coluna.dado.funcao"
            :class="
              (coluna.dado.classe || '') + (coluna.dado.funcao ? ' link ' : '')
            "
            :title="retornaDescricao(coluna.dado.descricao, linha)"
          >
            {{ retornaDados(linha, coluna) }}
          </td>
        </tr>
        <tr id="total" v-if="somar">
          <td v-if="seq" id="total" class="totais">TOTAL</td>
          <td
            v-for="coluna in myDataset.colunas"
            :key="columnId(coluna)"
            class="totais"
          >
            {{ retornaSoma(coluna) }}
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
export default {
  props: {
    seq: Boolean,
    somar: Boolean,
    id: String,
    dataSet: Object,
  },
  created() {
    if (this.dataSet) {
      let tmp = this.dataSet;
      $.each(tmp.origem, (chaveLinha, linha) => {
        $.each(tmp.colunas, (chaveColuna, coluna) => {
          if (coluna.total) {
            let tmpSoma = this.somas[coluna.dado.origem] || 0.0;
            tmpSoma += Number(this.returnValue(linha, coluna.dado.origem));
            this.somas[coluna.dado.origem] = tmpSoma;
          }
        });
      });
      this.myDataset = tmp;
    }
  },
  data() {
    return {
      myDataset: {},
      valorAnterior: [],
      ordem: [],
      somas: [],
    };
  },
  computed: {
    columnId() {
      return (coluna) => {
        return `${this.id}_${coluna.dado.origem}`;
      };
    },
    rowId() {
      return (linha) => {
        return `${this.id}_${linha.toString()}`;
      };
    },
    celId() {
      return (linha, coluna) => {
        return `${this.id}_${linha}_${coluna.dado.origem}`;
      };
    },
    retornaDados() {
      return (dados, coluna) => {
        let dado = "";
        let tipoDado = coluna.dado.tipoDado || "";
        let nomeColuna = coluna.dado.origem || "";
        let vlr;
        if (Boolean(dados[nomeColuna])) {
          switch (tipoDado) {
            case "data":
              let tmpdate = this.returnValue(dados, nomeColuna);
              let dt = new Date(this.returnValue(dados, nomeColuna));
              dado = Boolean(dt)
                ? `${tmpdate.substring(8, 10)}/${tmpdate.substring(
                    5
                  )}/${tmpdate.substring(0, 4)}`
                : "";
              break;
            case "moeda":
              vlr = parseFloat(this.returnValue(dados, nomeColuna)).toFixed(2);
              dado = formatarNumero(vlr, "pt-BR", "currency", "BRL", 2);
              break;
            case "valor":
            case "decimal":
              vlr = parseFloat(this.returnValue(dados, nomeColuna)).toFixed(2);
              dado = formatarNumero(vlr, "pt-BR", "decimal", "BRL", 2);
              break;
            case "inteiro":
              vlr = parseFloat(this.returnValue(dados, nomeColuna)).toFixed(0);
              dado = formatarNumero(vlr, "pt-BR", "decimal", "BRL", 0);
              break;
            default:
              dado = this.returnValue(dados, nomeColuna);
              break;
          }

          if (
            coluna.dado.naoRepetir &&
            dado == this.valorAnterior[nomeColuna]
          ) {
            dado = "";
          } else {
            this.valorAnterior[nomeColuna] = dado;
          }
          return dado;
        } else {
          return "";
        }
      };
    },
    retornaDescricao() {
      return (origemDados, linha) => {
        let retorno = "";
        let dados;

        if (typeof origemDados == "undefined") {
          return ``;
        }
        if (typeof origemDados == "string") {
          return `${origemDados}`;
        }
        if (Boolean(origemDados.origem)) {
          dados = origemDados.origem.filter((el) => {
            return el[origemDados.filtraEm] == linha[origemDados.filtraPor];
          });

          $.each(dados, (key, value) => {
            let linha = origemDados.descricao;
            $.each(value, (key, value) => {
              linha = linha.replace("{" + key + "}", value);
            });
            retorno +=
              origemDados.origem.length > 1 &&
              origemDados.origem.length != key + 1
                ? `${linha} \n`
                : `${linha} `;
          });
        }
        return retorno;
      };
    },
    retornaSoma() {
      return (coluna) => {
        let vlr;
        let dado = "0";
        if (coluna.total) {
          switch (coluna.dado.tipoDado) {
            case "moeda":
              vlr = parseFloat(this.somas[coluna.dado.origem]).toFixed(2);
              dado = isNaN(vlr)
                ? ""
                : formatarNumero(vlr, "pt-BR", "currency", "BRL", 2);
              break;
            case "valor":
            case "decimal":
              vlr = parseFloat(this.somas[coluna.dado.origem]).toFixed(2);
              dado = isNaN(vlr)
                ? ""
                : formatarNumero(vlr, "pt-BR", "decimal", "BRL", 2);
              break;
            case "inteiro":
              vlr = parseFloat(this.somas[coluna.dado.origem]).toFixed(0);
              dado = isNaN(vlr)
                ? ""
                : formatarNumero(vlr, "pt-BR", "decimal", "BRL", 0);
              break;
            default:
              dado = "";
              break;
          }
          return dado;
        } else {
          return ``;
        }
      };
    },
  },
  methods: {
    carregar() {
      let tmp = JSON.parse($(`#${this.id}`).attr("data-dataset"));
      $.each(tmp.origem, (chaveLinha, linha) => {
        $.each(tmp.colunas, (chaveColuna, coluna) => {
          if (coluna.total) {
            let tmpSoma = this.somas[coluna.dado.origem] || 0.0;
            tmpSoma += Number(this.returnValue(linha, coluna.dado.origem));
            this.somas[coluna.dado.origem] = tmpSoma;
          }
        });
      });
      this.myDataset = tmp;
    },
    limpar() {
      $(`#${this.id}>tbody tr`).remove();
    },
    returnValue(dados, campo) {
      let teste = campo.split(".");
      if (teste.length == 1) {
        return dados[teste[0]];
      } else {
        return this.returnValue(dados[teste[0]], teste[1]);
      }
    },
    reverseTabela(coluna) {
      if (typeof this.ordem[coluna] == "undefined") {
        this.ordem[coluna] = true;
      }
      if (this.ordem[coluna]) {
        this.myDataset.origem = this.sortByKey(this.myDataset.origem, coluna);
      } else {
        this.myDataset.origem = this.reverseByKey(
          this.myDataset.origem,
          coluna
        );
      }
      this.ordem[coluna] = !this.ordem[coluna];
      $(".ativo").attr(
        "class",
        $(".ativo").prop("class").replace("ativo", "").replace("  ", " ")
      );
      $(`#${this.id}_${coluna}`).attr("class", "ativo link");
    },
    sortByKey(array, key) {
      let tmp = [];
      $.each(array, (i, d) => {
        tmp.push(d);
      });

      return tmp.sort((a, b) => {
        let x = a[key];
        let y = b[key];

        return x < y ? -1 : x > y ? 1 : 0;
      });
    },
    reverseByKey(array, key) {
      let tmp = [];
      $.each(array, (i, d) => {
        tmp.push(d);
      });

      return tmp.sort((a, b) => {
        let x = a[key];
        let y = b[key];

        return x > y ? -1 : x < y ? 1 : 0;
      });
    },
  },
};

function formatarNumero(valor, location, style, currency, digitos) {
  var numero = new Intl.NumberFormat(location, {
    style: style,
    currency: currency,
    minimumFractionDigits: digitos,
    maximumFractionDigits: digitos,
  });

  return numero.format(valor);
}
</script>

<style lang="css">
table {
  background-color: aliceblue;
}

.link {
  cursor: pointer;
}
.ativo {
  border: 2px solid !important ;
}
.totais {
  font-weight: bold;
}
</style>