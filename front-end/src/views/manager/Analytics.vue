<template>
    <div class="inventory-wrapper my-8">
      <div class="container mx-auto p-4">
        <div class="datepicker-container flex justify-center gap-4 mb-4">
          <Datepicker class="datepicker form-input" v-model="startDate" model-type="yyyy-MM-dd" placeholder="Select Start Date" :enable-time-picker="false" :disabled="isRestockReportSelected"/>
          <Datepicker class="datepicker form-input" v-model="endDate" model-type="yyyy-MM-dd" placeholder="Select End Date" :enable-time-picker="false" :disabled="isExcessReportSelected || isRestockReportSelected"/>
        </div>

        <div class="dropdown relative mb-4">
  <button class="dropdown-toggle bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600 focus:outline-none focus:ring-2 focus:ring-green-700" @click="toggleDropdown">
    {{ selectedOption.name || 'Select A Query' }}
  </button>
  <div class="dropdown-menu absolute w-full bg-white shadow-lg rounded mt-1 z-10" v-show="isOpen">
    <a class="dropdown-item p-2 hover:bg-gray-100 block text-gray-800" v-for="option in options" :key="option.name" @click="selectOption(option)">
      {{ option.name }}
    </a>
  </div>
</div>

        <p class="description text-lg text-gray-700 mb-4"> {{ selectedOption.description }}</p>
        <button class="run-button bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded" v-if="isSelected" @click="selectedOption.fetch">Run</button>


        <div v-if="selectedOption.type === '2'" class="relative overflow-x-auto shadow-lg rounded-lg">

            <table class="w-full text-sm text-left text-gray-900">
                <thead class="text-sm text-red-700 uppercase bg-red-100">
                    <tr>
                        <th scope="col" class="px-6 py-3">
                            {{ selectedOption.col_1  }}
                        </th>
                    
                        <th scope="col" class="px-6 py-3">
                            {{ selectedOption.col_2  }}
                        </th>
                    </tr>
                </thead>
                <tbody> <!-- start auto populating here -->
                    <tr v-if="paginatedData.length === 0">
                            <td colspan="5" class="text-center py-4">No items match your search.</td>
                        </tr>
                    <tr v-else v-for="product in paginatedData" :key="product.id">
                        <th scope="row" class="px-6 py-4 font-medium text-gray-900">
                            {{ product.name }} <!-- name -->
                        </th>
                        <td class="px-6 py-4">
                            {{ product.count }} <!-- quantity -->
                        </td>
                    </tr>
                </tbody>
            </table>
            <div class="flex justify-between items-center p-4 text-md text-red-700 uppercase bg-gray-50">
                <button @click="currentPage > 1 && currentPage--" :disabled="currentPage <= 1">Previous</button>
                <span>Page {{ currentPage }} of {{ totalPages }}</span>
                <button @click="currentPage < totalPages && currentPage++" :disabled="currentPage >= totalPages">Next</button>
            </div>
        </div>
        <div v-if="selectedOption.type === '3'" class="relative overflow-x-auto shadow-lg rounded-lg">

            <table class="w-full text-sm text-left text-gray-900">
                <thead class="text-sm text-red-700 uppercase bg-red-100">
                    <tr>
                        <th scope="col" class="px-6 py-3">
                            {{ selectedOption.col_1 }}
                        </th>
                        <th scope="col" class="px-6 py-3">
                            {{ selectedOption.col_2 }}
                        </th>
                        <th scope="col" class="px-6 py-3">
                            {{ selectedOption.col_3 }}
                        </th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-if="paginatedData.length === 0">
                            <td colspan="5" class="text-center py-4">No items match your search.</td>
                        </tr>
                    <tr v-else v-for="product in paginatedData" :key="product.id">
                        <th scope="row" class="px-6 py-4 font-medium text-gray-900">
                            {{ product.element1 }}
                        </th>
                        <td class="px-6 py-4">
                            {{ product.element2 }}
                        </td>
                        <td class="px-6 py-4">
                            {{ product.element3 }}
                        </td>
                    </tr>
                </tbody>
            </table>
            <div class="flex justify-between items-center p-4 text-md text-red-700 uppercase bg-gray-50">
                <button @click="currentPage > 1 && currentPage--" :disabled="currentPage <= 1">Previous</button>
                <span>Page {{ currentPage }} of {{ totalPages }}</span>
                <button @click="currentPage < totalPages && currentPage++" :disabled="currentPage >= totalPages">Next</button>
            </div>

        </div>
    </div>
    
    </div>
</template>

<script>
  import { ref } from 'vue';
  import Datepicker from '@vuepic/vue-datepicker';
  import '@vuepic/vue-datepicker/dist/main.css';
  import TableDropdown from "@/components/Dropdowns/TableDropdown.vue";
  import axios from 'axios';
  import { MenuCategories, PROD_BASE_URL } from '../../main';
  
  export default {
    components: {
        Datepicker,
        TableDropdown
    },
    props: {
        color: {
            default: "light",
            validator: function (value) {
                return ["light", "dark"].indexOf(value) !== -1;
            },
        },
    },
    data() {
        return {
            currentPage: 1,
            perPage: 10,
            searchQuery: '',

            productUsage: [],
            salesReport: [],
            excessItems: [],
            restockReport: [],
            sellsTogethor: [],
            startDate: ref(""),
            endDate: ref(""),
            isOpen: false,
            isSelected: false,
            selectedOption: { name: null, type: null, col_1: null, col_2: null, fetch: null },
            options: [
                { name: 'Product Usage Table', type: '2', col_1: 'Item Name', col_2: 'Quantity Used', fetch: this.fetchProductUsage, description: 'Given a time window it displays the number of each inventory item used during that time period.' }, // will use .name and .count to get the elements out of array 
                { name: 'Sales Report', type: '3', col_1: 'Item Name', col_2: 'Quantity Sold', col_3: 'Revenue', fetch: this.fetchSalesReport, description: 'Given a time window it displays the number and revenue of each menu item sold during that time period.' }, // will use .element1 , .element2 , .element3 
                { name: 'Excess Report', type: '3', col_1: 'Item Name', col_2: 'Quantity Used', col_3: 'Current Stock', fetch: this.fetchExcessItems, description: 'Given a timestamp, it displays the list of inventory items that sold less than 10% of their quantity between the timestamp and the current time.'},
                { name: 'Restock Report', type: '3', col_1: 'Item Name', col_2: 'Current Amount', col_3: 'Restock Level', fetch: this.fetchRestockReport, description: 'Displays a list of inventory items whose current stock is less than the items reorder level'},
                { name: 'Common Pairings', type: '3', col_1: 'Item Pair 1', col_2: 'Item Pair 2', col_3: 'Pairings', fetch: this.fetchSellsTogethor, description: 'Given a time window, it displays a list of pairs of menu items that sell together, sorted by most frequent.'},
            ],
        }   
    },
    computed: {
        selectedArray() {
            console.log('Selected Option:', this.selectedOption.name);
            switch (this.selectedOption.name) {
                case 'Product Usage Table':
                    return this.productUsage;
                case 'Sales Report':
                    return this.salesReport;
                case 'Excess Report':
                    return this.excessItems;
                case 'Restock Report':
                    return this.restockReport;
                case 'Common Pairings':
                    return this.sellsTogethor;
                default:
                    return [];
            }
        },
        isExcessReportSelected() {
            return this.selectedOption && this.selectedOption.name === 'Excess Report';
        },
        isRestockReportSelected() {
            return this.selectedOption && this.selectedOption.name === 'Restock Report';
        },
        totalPages() {
            return Math.ceil(this.selectedArray.length / this.perPage);
        },
        paginatedData() {
            const start = (this.currentPage - 1) * this.perPage;
            const end = start + this.perPage;
            return this.selectedArray.slice(start, end);
        },
    },  
    methods: {
        nextPage() {
            if (this.currentPage < this.totalPages) {
                this.currentPage++;
            }
        },
        prevPage() {
            if (this.currentPage > 1) {
                this.currentPage--;
            }
        },


        fetchProductUsage() {
            axios.get(`${PROD_BASE_URL}/product_usage_chart_data/`, {
                params: {
                    // "2022-03-29 00:00:00"
                    // "2024-03-28 23:59:59"
                startDate: this.startDate.concat(" 00:00:00"),
                endDate: this.endDate.concat(" 23:59:59")
                }
            })
            .then(response => {
                this.productUsage = response.data;
            })
            .catch(error => {
                console.error('Error:', error);
            });
            this.isSelected = false;
        },
        fetchSalesReport() {
            axios.get(`${PROD_BASE_URL}/sales_report/`, {
                params: {
                    // "2022-03-29 00:00:00"
                    // "2024-03-28 23:59:59"
                startDate: this.startDate.concat(" 00:00:00"),
                endDate: this.endDate.concat(" 23:59:59")
                }
            })
            .then(response => {
                this.salesReport = response.data;
            })
            .catch(error => {
                console.error('Error:', error);
            });
            this.isSelected = false;
        },
        fetchExcessItems() {
            axios.get(`${PROD_BASE_URL}/excess_report/`, {
                params: {
                    // "2024-03-28 23:59:59"
                endDate: this.endDate.concat(" 23:59:59")
                }
            })
            .then(response => {
                this.excessItems = response.data;
            })
            .catch(error => {
                console.error('Error:', error);
            });
            this.isSelected = false;
        },
        fetchRestockReport() {
            axios.get(`${PROD_BASE_URL}/restock_report/`)
            .then(response => {
                this.restockReport = response.data;
            })
            .catch(error => {
                console.error('Error:', error);
            });
            this.isSelected = false;
        },
        fetchSellsTogethor() {
            axios.get(`${PROD_BASE_URL}/sells_together/`, {
                params: {
                    // "2022-03-29 00:00:00"
                    // "2024-03-28 23:59:59"
                startDate: this.startDate.concat(" 00:00:00"),
                endDate: this.endDate.concat(" 23:59:59")
                }
            })
            .then(response => {
                this.sellsTogethor = response.data;
            })
            .catch(error => {
                console.error('Error:', error);
            });
            this.isSelected = false;
        },
        toggleDropdown() {
            this.isOpen = !this.isOpen;
        },
        selectOption(option) {
            this.selectedOption = option;
            this.isOpen = false;
            this.isSelected = true;
            this.currentPage = 1;
        },
    }
  }
</script>

<style>
body {
    color: black;
    background-color: #580726; /* This dones not work for some reason */
}


.disabled-datepicker {
  background-color: #a0a4ab;
  color: #52575c; 
  cursor: not-allowed;
}


</style>

