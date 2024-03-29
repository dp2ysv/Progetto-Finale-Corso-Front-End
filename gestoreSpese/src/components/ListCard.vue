<script>
import CreateItem from "./CreateItem.vue";
import EditItem from "./EditItem.vue";
import { jsPDF } from "jspdf";
import lunr from "lunr";
import html2canvas from "html2canvas";
import Papa from "papaparse";
import ExpenseChart from './ExpenseChart.vue';

export default {
  props: ["list", "totale"],
  data() {
    return {
      showEdit: false,
      showCreate: false,
      confirmDelete: false,
      filter: false,
      menu: false,
      filteredList: [],
      filteredSomma: 0,
      categoria: "Seleziona Categoria",
      itemToDelete: "",
      itemToEdit: "",
      idx: null, // Aggiungi  per memorizzare l'indice Lunr.js
      searchQuery: "", // Aggiungi  per memorizzare la query di ricerca
    };
  },
  components: {
    CreateItem,
    EditItem,
    ExpenseChart
  },

  mounted() {
    this.createSearchIndex(); // Chiama la funzione per creare l'indice di ricerca al montaggio del componente
  },

  methods: {
    createNew() {
      this.showCreate = !this.showCreate;
    },

    showMenu() {
      this.menu = !this.menu;
    },

    setCategoria(categoria) {
      this.categoria = categoria;
      this.filter = true;
      this.menu = !this.menu;
      this.filteredList = this.list.filter(
        (element) => element.categoria == this.categoria
      );
      this.filteredSomma = 0;
      this.filteredList.forEach((element) => {
        this.filteredSomma += element.importo;
      });
      console.log(this.filteredSomma);
    },

    resetCategoria() {
      this.categoria = "Seleziona Categoria";
      this.filter = false;
    },

    showDelete(id) {
      this.confirmDelete = !this.confirmDelete;
      this.itemToDelete = id;
    },

    removeItem() {
      const index = this.list.findIndex(
        (item) => item.id === this.itemToDelete
      );
      if (index !== -1) {
        this.list.splice(index, 1);
        localStorage.setItem("item", JSON.stringify(this.list));
        this.confirmDelete = !this.confirmDelete;
      }
    },

    setEdit(id) {
      this.showEdit = !this.showEdit;
      const index = this.list.findIndex((item) => item.id === id);
      if (index !== -1) {
        this.itemToEdit = this.list[index];
      }
    },

    exportToPDF() {
      const element = this.$refs.tableToExport;
      html2canvas(element).then((canvas) => {
        const imgData = canvas.toDataURL("image/png");
        const pdf = new jsPDF({
          orientation: "landscape",
          unit: "pt",
          format: [canvas.width, canvas.height],
        });

        pdf.addImage(imgData, "PNG", 0, 0, canvas.width, canvas.height);
        pdf.save("spese.pdf");
      });
    },

    createSearchIndex() {
      let lista = this.list;
      this.idx = lunr(function () {
        this.ref("id");
        this.field("categoria");
        this.field("descrizione");
        this.field("importo");
        lista.forEach((item) => {
          this.add(item);
        });
      });
    },

    searchItems() {
      if (!this.idx || !this.searchQuery) {
        this.filter = false;
        return;
      }
      this.filteredList = this.idx
        .search(this.searchQuery)
        .map(({ ref }) => this.list.find((item) => item.id.toString() === ref));
      this.filteredSomma = 0;
      this.filteredList.forEach((element) => {
        this.filteredSomma += element.importo;
      });
      this.filter = true; // Attiva il filtro per mostrare i risultati della ricerca
    },

    exportToCSVWithPapaParse() {
      const csv = Papa.unparse(this.list);
      const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
      const link = document.createElement("a");
      const url = URL.createObjectURL(blob);
      link.setAttribute("href", url);
      link.setAttribute("download", "spese.csv");
      link.style.visibility = "hidden";
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    },
  },
};
</script>

<template>
  <CreateItem v-show="showCreate" />
  <EditItem v-show="showEdit" :item="itemToEdit"></EditItem>
  <div class="relative flex justify-between items-center sm:px-5 sm:pt-2">
    <div>
        <div class="flex flex-col-reverse justify-center items-center md:gap-4 md:flex-row">
          <div class="flex justify-center items-center gap-2 ml-4 mb-2 md:gap-4 md:mb-0">
            <button
            @click="showMenu"
            data-dropdown-toggle="dropdown"
            class="text-white focus:ring-4 focus:outline-none font-medium rounded-lg text-sm px-5 py-2.5 text-center inline-flex items-center bg-gray-800 hover:bg-gray-700 focus:ring-gray-300"
            type="button"
                          >
                          {{ categoria }}
                          <svg
                           class="w-2.5 h-2.5 ms-3"
                           aria-hidden="true"
                            xmlns="http://www.w3.org/2000/svg"
                             fill="none"
                              viewBox="0 0 10 6"
                                  >
                                  <path
                                  stroke="currentColor"
                                  stroke-linecap="round"
                                  stroke-linejoin="round"
                                  stroke-width="2"
                                d="m1 1 4 4 4-4"
                                      />
                                  </svg>
              </button> 
              <a
              v-show="filter"
                class="text-white cursor-pointer"
                @click="resetCategoria"
                ><svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
              <path stroke-linecap="round" stroke-linejoin="round" d="m9.75 9.75 4.5 4.5m0-4.5-4.5 4.5M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z" />
              </svg>
            
              </a>
            </div>
              <input
                v-model="searchQuery"
                @input="searchItems"
                  placeholder="Ricerca"
                  class="block h-auto w-50 rounded-lg p-2 text-sm my-2 text-gray-900 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
                  />
          </div>
      </div>

        <button
      @click="createNew"
      type="button"
      class="text-white m-4 bg-gray-800 hover:bg-gray-700 focus:outline-none focus:ring-4 focus:ring-gray-300 font-medium rounded-lg text-sm px-5 py-2.5 me-2 mb-2 dark:bg-gray-800 dark:hover:bg-gray-700 dark:focus:ring-gray-700 dark:border-gray-700"
        >
        <span v-if="!showCreate">Aggiungi Nuovo</span> <span v-else>Chiudi</span>
        </button>
    </div>

  <!-- Menu dropdown -->
  <div v-show="menu" class="absolute pl-[22px] md:pl-[42px]">
    <div
      class="z-10 bg-white divide-y divide-gray-100 rounded-lg shadow w-44 dark:bg-gray-700"
    >
      <ul
        class="py-2 text-sm text-gray-700 dark:text-gray-200"
        aria-labelledby="dropdownDefaultButton"
      >
        <li v-for="item in list" :key="item.categoria">
          <a
            @click="setCategoria(item.categoria)"
            class="block px-4 py-2 hover:bg-gray-100 dark:hover:bg-gray-600 dark:hover:text-white"
            >{{ item.categoria }}</a
          >
        </li>
      </ul>
    </div>
  </div>

  <div class="mx-4 overflow-x-auto shadow-md sm:rounded-lg">
    <table
      ref="tableToExport"
      class="w-full text-sm text-left rtl:text-right text-gray-600 dark:text-gray-400"
    >
      <thead class="text-sm uppercase bg-gray-700 text-gray-400">
        <tr>
          <th scope="col" class="px-6 py-3">Data</th>
          <th scope="col" class="px-6 py-3">Categoria</th>
          <th scope="col" class="px-6 py-3">Descrizione</th>
          <th scope="col" class="px-6 py-3">Importo</th>
          <th scope="col" class="px-6 py-3">Azioni</th>
        </tr>
      </thead>
      <tbody v-if="list && !filter" v-for="item in list" :key="item.id">
        <tr class="border-b bg-gray-800 border-gray-700">
          <th
            scope="row"
            class="px-6 py-4 font-normal whitespace-nowrap text-white text-[17px]"
          >
            {{ item.data }}
          </th>
          <td class="px-6 py-4 text-[17px] text-white">{{ item.categoria }}</td>
          <td class="px-6 py-4 text-[17px] text-white">
            {{ item.descrizione }}
          </td>
          <td class="px-6 py-4 text-[17px] text-white">
            € {{ parseFloat(item.importo).toFixed(2) }}
          </td>
          <td class="px-6 py-4 text-[17px] text-white">
            <button
              @click="setEdit(item.id)"
              class="font-medium mr-4 text-blue-500 hover:underline"
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                stroke-width="1.5"
                stroke="currentColor"
                class="w-6 h-6"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="m16.862 4.487 1.687-1.688a1.875 1.875 0 1 1 2.652 2.652L10.582 16.07a4.5 4.5 0 0 1-1.897 1.13L6 18l.8-2.685a4.5 4.5 0 0 1 1.13-1.897l8.932-8.931Zm0 0L19.5 7.125M18 14v4.75A2.25 2.25 0 0 1 15.75 21H5.25A2.25 2.25 0 0 1 3 18.75V8.25A2.25 2.25 0 0 1 5.25 6H10"
                />
              </svg>
            </button>
            <button
              @click="showDelete(item.id)"
              class="font-medium mr-4 text-red-500 hover:underline"
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                stroke-width="1.5"
                stroke="currentColor"
                class="w-6 h-6"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="m14.74 9-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 0 1-2.244 2.077H8.084a2.25 2.25 0 0 1-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 0 0-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 0 1 3.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 0 0-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 0 0-7.5 0"
                />
              </svg>
            </button>

            <div
              v-show="confirmDelete"
              class="flex justify-center items-center overflow-y-auto overflow-x-hidden fixed top-0 right-0 left-0 z-50 w-full md:inset-0 h-[calc(100%-1rem)] max-h-full"
            >
              <div class="relative p-4 w-full max-w-md max-h-full">
                <div class="relative rounded-lg shadow bg-gray-700">
                  <button
                    @click="showDelete"
                    type="button"
                    class="absolute top-3 end-2.5 bg-transparent rounded-lg text-sm w-8 h-8 ms-auto inline-flex justify-center items-center hover:bg-gray-600 hover:text-white"
                    data-modal-hide="popup-modal"
                  >
                    <svg
                      class="w-3 h-3"
                      aria-hidden="true"
                      xmlns="http://www.w3.org/2000/svg"
                      fill="none"
                      viewBox="0 0 14 14"
                    >
                      <path
                        stroke="currentColor"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="m1 1 6 6m0 0 6 6M7 7l6-6M7 7l-6 6"
                      />
                    </svg>
                    <span class="sr-only">Close modal</span>
                  </button>
                  <div class="p-4 md:p-5 text-center">
                    <svg
                      class="mx-auto mb-4 w-12 h-12 text-gray-200"
                      aria-hidden="true"
                      xmlns="http://www.w3.org/2000/svg"
                      fill="none"
                      viewBox="0 0 20 20"
                    >
                      <path
                        stroke="currentColor"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="M10 11V6m0 8h.01M19 10a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z"
                      />
                    </svg>
                    <h3 class="mb-5 text-lg font-normal text-gray-400">
                      Are you sure you want to delete?
                    </h3>
                    <button
                      @click="removeItem"
                      type="button"
                      class="text-white bg-red-600 hover:bg-red-800 focus:ring-4 focus:outline-none focus:ring-red-800 font-medium rounded-lg text-sm inline-flex items-center px-5 py-2.5 text-center"
                    >
                      Yes, I'm sure
                    </button>
                    <button
                      @click="showDelete"
                      data-modal-hide="popup-modal"
                      type="button"
                      class="py-2.5 px-5 ms-3 text-sm font-medium focus:outline-none rounded-lg border focus:z-10 focus:ring-4 focus:ring-gray-700 bg-gray-800 text-gray-400 border-gray-600 hover:text-white hover:bg-gray-700"
                    >
                      No, cancel
                    </button>
                  </div>
                </div>
              </div>
            </div>
          </td>
        </tr>
      </tbody>

      <tbody v-if="filter" v-for="item in filteredList" :key="item.id">
        <tr class="border-b bg-gray-800 border-gray-700">
          <th
            scope="row"
            class="px-6 py-4 font-normal whitespace-nowrap text-white text-[17px]"
          >
            {{ item.data }}
          </th>
          <td class="px-6 py-4 text-[17px] text-white">{{ item.categoria }}</td>
          <td class="px-6 py-4 text-[17px] text-white">
            {{ item.descrizione }}
          </td>
          <td class="px-6 py-4 text-[17px] text-white">
            € {{ parseFloat(item.importo).toFixed(2) }}
          </td>
          <td class="px-6 py-4 text-[17px] text-white">
            <button
              @click="setEdit(item.id)"
              class="font-medium mr-4 text-blue-500 hover:underline"
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                stroke-width="1.5"
                stroke="currentColor"
                class="w-6 h-6"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="m16.862 4.487 1.687-1.688a1.875 1.875 0 1 1 2.652 2.652L10.582 16.07a4.5 4.5 0 0 1-1.897 1.13L6 18l.8-2.685a4.5 4.5 0 0 1 1.13-1.897l8.932-8.931Zm0 0L19.5 7.125M18 14v4.75A2.25 2.25 0 0 1 15.75 21H5.25A2.25 2.25 0 0 1 3 18.75V8.25A2.25 2.25 0 0 1 5.25 6H10"
                />
              </svg>
            </button>
            <button
              @click="showDelete(item.id)"
              class="font-medium mr-4 text-red-500 hover:underline"
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
                stroke-width="1.5"
                stroke="currentColor"
                class="w-6 h-6"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="m14.74 9-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 0 1-2.244 2.077H8.084a2.25 2.25 0 0 1-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 0 0-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 0 1 3.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 0 0-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 0 0-7.5 0"
                />
              </svg>
            </button>

            <div
              v-show="confirmDelete"
              class="flex justify-center items-center overflow-y-auto overflow-x-hidden fixed top-0 right-0 left-0 z-50 w-full md:inset-0 h-[calc(100%-1rem)] max-h-full"
            >
              <div class="relative p-4 w-full max-w-md max-h-full">
                <div class="relative rounded-lg shadow bg-gray-700">
                  <button
                    @click="showDelete"
                    type="button"
                    class="absolute top-3 end-2.5 bg-transparent rounded-lg text-sm w-8 h-8 ms-auto inline-flex justify-center items-center hover:bg-gray-600 hover:text-white"
                    data-modal-hide="popup-modal"
                  >
                    <svg
                      class="w-3 h-3"
                      aria-hidden="true"
                      xmlns="http://www.w3.org/2000/svg"
                      fill="none"
                      viewBox="0 0 14 14"
                    >
                      <path
                        stroke="currentColor"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="m1 1 6 6m0 0 6 6M7 7l6-6M7 7l-6 6"
                      />
                    </svg>
                    <span class="sr-only">Close modal</span>
                  </button>
                  <div class="p-4 md:p-5 text-center">
                    <svg
                      class="mx-auto mb-4 w-12 h-12 text-gray-200"
                      aria-hidden="true"
                      xmlns="http://www.w3.org/2000/svg"
                      fill="none"
                      viewBox="0 0 20 20"
                    >
                      <path
                        stroke="currentColor"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="M10 11V6m0 8h.01M19 10a9 9 0 1 1-18 0 9 9 0 0 1 18 0Z"
                      />
                    </svg>
                    <h3 class="mb-5 text-lg font-normal text-gray-400">
                      Are you sure you want to delete?
                    </h3>
                    <button
                      @click="removeItem"
                      type="button"
                      class="text-white bg-red-600 hover:bg-red-800 focus:ring-4 focus:outline-none focus:ring-red-800 font-medium rounded-lg text-sm inline-flex items-center px-5 py-2.5 text-center"
                    >
                      Yes, I'm sure
                    </button>
                    <button
                      @click="showDelete"
                      data-modal-hide="popup-modal"
                      type="button"
                      class="py-2.5 px-5 ms-3 text-sm font-medium focus:outline-none rounded-lg border focus:z-10 focus:ring-4 focus:ring-gray-700 bg-gray-800 text-gray-400 border-gray-600 hover:text-white hover:bg-gray-700"
                    >
                      No, cancel
                    </button>
                  </div>
                </div>
              </div>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="flex justify-between p-4 min-w-8 mx-auto">
    <div>
      <button
        @click="exportToPDF"
        type="button"
        class="text-white m-4  hover:bg-gray-700 focus:outline-none focus:ring-4 focus:ring-gray-300 font-medium rounded-lg text-sm px-5 py-2.5 dark:bg-blue-600 dark:hover:bg-blue-700 dark:focus:ring-blue-800"
      ><svg height="40px" width="40px" version="1.1" id="Layer_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" 
	 viewBox="0 0 303.188 303.188" xml:space="preserve">
<g>
	<polygon style="fill:#E8E8E8;" points="219.821,0 32.842,0 32.842,303.188 270.346,303.188 270.346,50.525 	"/>
	<path style="fill:#FB3449;" d="M230.013,149.935c-3.643-6.493-16.231-8.533-22.006-9.451c-4.552-0.724-9.199-0.94-13.803-0.936
		c-3.615-0.024-7.177,0.154-10.693,0.354c-1.296,0.087-2.579,0.199-3.861,0.31c-1.314-1.36-2.584-2.765-3.813-4.202
		c-7.82-9.257-14.134-19.755-19.279-30.664c1.366-5.271,2.459-10.772,3.119-16.485c1.205-10.427,1.619-22.31-2.288-32.251
		c-1.349-3.431-4.946-7.608-9.096-5.528c-4.771,2.392-6.113,9.169-6.502,13.973c-0.313,3.883-0.094,7.776,0.558,11.594
		c0.664,3.844,1.733,7.494,2.897,11.139c1.086,3.342,2.283,6.658,3.588,9.943c-0.828,2.586-1.707,5.127-2.63,7.603
		c-2.152,5.643-4.479,11.004-6.717,16.161c-1.18,2.557-2.335,5.06-3.465,7.507c-3.576,7.855-7.458,15.566-11.815,23.02
		c-10.163,3.585-19.283,7.741-26.857,12.625c-4.063,2.625-7.652,5.476-10.641,8.603c-2.822,2.952-5.69,6.783-5.941,11.024
		c-0.141,2.394,0.807,4.717,2.768,6.137c2.697,2.015,6.271,1.881,9.4,1.225c10.25-2.15,18.121-10.961,24.824-18.387
		c4.617-5.115,9.872-11.61,15.369-19.465c0.012-0.018,0.024-0.036,0.037-0.054c9.428-2.923,19.689-5.391,30.579-7.205
		c4.975-0.825,10.082-1.5,15.291-1.974c3.663,3.431,7.621,6.555,11.939,9.164c3.363,2.069,6.94,3.816,10.684,5.119
		c3.786,1.237,7.595,2.247,11.528,2.886c1.986,0.284,4.017,0.413,6.092,0.335c4.631-0.175,11.278-1.951,11.714-7.57
		C231.127,152.765,230.756,151.257,230.013,149.935z M119.144,160.245c-2.169,3.36-4.261,6.382-6.232,9.041
		c-4.827,6.568-10.34,14.369-18.322,17.286c-1.516,0.554-3.512,1.126-5.616,1.002c-1.874-0.11-3.722-0.937-3.637-3.065
		c0.042-1.114,0.587-2.535,1.423-3.931c0.915-1.531,2.048-2.935,3.275-4.226c2.629-2.762,5.953-5.439,9.777-7.918
		c5.865-3.805,12.867-7.23,20.672-10.286C120.035,158.858,119.587,159.564,119.144,160.245z M146.366,75.985
		c-0.602-3.514-0.693-7.077-0.323-10.503c0.184-1.713,0.533-3.385,1.038-4.952c0.428-1.33,1.352-4.576,2.826-4.993
		c2.43-0.688,3.177,4.529,3.452,6.005c1.566,8.396,0.186,17.733-1.693,25.969c-0.299,1.31-0.632,2.599-0.973,3.883
		c-0.582-1.601-1.137-3.207-1.648-4.821C147.945,83.048,146.939,79.482,146.366,75.985z M163.049,142.265
		c-9.13,1.48-17.815,3.419-25.979,5.708c0.983-0.275,5.475-8.788,6.477-10.555c4.721-8.315,8.583-17.042,11.358-26.197
		c4.9,9.691,10.847,18.962,18.153,27.214c0.673,0.749,1.357,1.489,2.053,2.22C171.017,141.096,166.988,141.633,163.049,142.265z
		 M224.793,153.959c-0.334,1.805-4.189,2.837-5.988,3.121c-5.316,0.836-10.94,0.167-16.028-1.542
		c-3.491-1.172-6.858-2.768-10.057-4.688c-3.18-1.921-6.155-4.181-8.936-6.673c3.429-0.206,6.9-0.341,10.388-0.275
		c3.488,0.035,7.003,0.211,10.475,0.664c6.511,0.726,13.807,2.961,18.932,7.186C224.588,152.585,224.91,153.321,224.793,153.959z"/>
	<polygon style="fill:#FB3449;" points="227.64,25.263 32.842,25.263 32.842,0 219.821,0 	"/>
	<g>
		<path style="fill:#A4A9AD;" d="M126.841,241.152c0,5.361-1.58,9.501-4.742,12.421c-3.162,2.921-7.652,4.381-13.472,4.381h-3.643
			v15.917H92.022v-47.979h16.606c6.06,0,10.611,1.324,13.652,3.971C125.321,232.51,126.841,236.273,126.841,241.152z
			 M104.985,247.387h2.363c1.947,0,3.495-0.546,4.644-1.641c1.149-1.094,1.723-2.604,1.723-4.529c0-3.238-1.794-4.857-5.382-4.857
			h-3.348C104.985,236.36,104.985,247.387,104.985,247.387z"/>
		<path style="fill:#A4A9AD;" d="M175.215,248.864c0,8.007-2.205,14.177-6.613,18.509s-10.606,6.498-18.591,6.498h-15.523v-47.979
			h16.606c7.701,0,13.646,1.969,17.836,5.907C173.119,235.737,175.215,241.426,175.215,248.864z M161.76,249.324
			c0-4.398-0.87-7.657-2.609-9.78c-1.739-2.122-4.381-3.183-7.926-3.183h-3.773v26.877h2.888c3.939,0,6.826-1.143,8.664-3.43
			C160.841,257.523,161.76,254.028,161.76,249.324z"/>
		<path style="fill:#A4A9AD;" d="M196.579,273.871h-12.766v-47.979h28.355v10.403h-15.589v9.156h14.374v10.403h-14.374
			L196.579,273.871L196.579,273.871z"/>
	</g>
	<polygon style="fill:#D1D3D3;" points="219.821,50.525 270.346,50.525 219.821,0 	"/>
</g>
</svg>
      </button>
      <button
        @click="exportToCSVWithPapaParse"
        type="button"
        class="text-white  hover:bg-gray-700 focus:outline-none focus:ring-4 focus:ring-gray-300 font-medium rounded-lg text-sm px-5 py-2.5 dark:bg-blue-600 dark:hover:bg-blue-700 dark:focus:ring-blue-800"
      ><svg width="40px" height="40px" viewBox="-4 0 64 64" xmlns="http://www.w3.org/2000/svg">

<path d="M5.106 0c-2.802 0-5.073 2.272-5.073 5.074v53.841c0 2.803 2.271 5.074 5.073 5.074h45.774c2.801 0 5.074-2.271 5.074-5.074v-38.605l-18.903-20.31h-31.945z" fill-rule="evenodd" clip-rule="evenodd" fill="#45B058"/>

<path d="M20.306 43.197c.126.144.198.324.198.522 0 .378-.306.72-.703.72-.18 0-.378-.072-.504-.234-.702-.846-1.891-1.387-3.007-1.387-2.629 0-4.627 2.017-4.627 4.88 0 2.845 1.999 4.879 4.627 4.879 1.134 0 2.25-.486 3.007-1.369.125-.144.324-.233.504-.233.415 0 .703.359.703.738 0 .18-.072.36-.198.504-.937.972-2.215 1.693-4.015 1.693-3.457 0-6.176-2.521-6.176-6.212s2.719-6.212 6.176-6.212c1.8.001 3.096.721 4.015 1.711zm6.802 10.714c-1.782 0-3.187-.594-4.213-1.495-.162-.144-.234-.342-.234-.54 0-.361.27-.757.702-.757.144 0 .306.036.432.144.828.739 1.98 1.314 3.367 1.314 2.143 0 2.827-1.152 2.827-2.071 0-3.097-7.112-1.386-7.112-5.672 0-1.98 1.764-3.331 4.123-3.331 1.548 0 2.881.467 3.853 1.278.162.144.252.342.252.54 0 .36-.306.72-.703.72-.144 0-.306-.054-.432-.162-.882-.72-1.98-1.044-3.079-1.044-1.44 0-2.467.774-2.467 1.909 0 2.701 7.112 1.152 7.112 5.636.001 1.748-1.187 3.531-4.428 3.531zm16.994-11.254l-4.159 10.335c-.198.486-.685.81-1.188.81h-.036c-.522 0-1.008-.324-1.207-.81l-4.142-10.335c-.036-.09-.054-.18-.054-.288 0-.36.323-.793.81-.793.306 0 .594.18.72.486l3.889 9.992 3.889-9.992c.108-.288.396-.486.72-.486.468 0 .81.378.81.793.001.09-.017.198-.052.288z" fill="#ffffff"/>

<g fill-rule="evenodd" clip-rule="evenodd">

<path d="M56.001 20.357v1h-12.8s-6.312-1.26-6.128-6.707c0 0 .208 5.707 6.003 5.707h12.925z" fill="#349C42"/>

<path d="M37.098.006v14.561c0 1.656 1.104 5.791 6.104 5.791h12.8l-18.904-20.352z" opacity=".5" fill="#ffffff"/>

</g>

</svg>
      </button>
    </div>
    <div
      v-show="filter"
      class="my-4 text-white focus:ring-4 focus:outline-none font-medium rounded-lg shadow-2xl text-sm px-5 py-2.5 text-center flex justify-center items-center bg-gray-800 hover:bg-gray-700 focus:ring-gray-800"
    >
      {{ "Spesa Filtrata = € " + parseFloat(filteredSomma).toFixed(2) }}
    </div>
    <div
      v-show="!filter"
      class="my-4 text-white focus:ring-4 focus:outline-none font-medium rounded-lg shadow-2xl text-sm px-5 py-2.5 text-center flex justify-center items-center bg-gray-800 hover:bg-gray-700 focus:ring-gray-800"
    >
      {{ "Spesa Totale " + "= " + "€ " + parseFloat(totale).toFixed(2) }}
    </div>
  </div>
  
</template>
