<template>
  <div>
    <button @click="redirectToViewWeeklyProgressReport" class="redirect-btn">
      View Weekly Progress Report
    </button>
    <button @click="downloadPDF" class="download-btn">
      Download as PDF
    </button>
    <!-- Responsive container with side margins -->
    <div class="table-container">
      <table ref="pdfTable" border="1" class="mx-auto my-5 pdf-table">
        <thead>
          <tr>
            <th colspan="16">
              <img src="@/assets/h0.png" alt="Header Image" class="header-image">
            </th>
          </tr>
        </thead>
        <thead>
          <tr>
            <th rowspan="2">ITEM NO.</th>
            <th rowspan="2">DESCRIPTION</th>
            <th colspan="4">ORIGINAL CONTRACT</th>
            <th colspan="10">WORK ACCOMPLISHED</th>
          </tr>
          <tr>
            <!-- ORIGINAL CONTRACT columns -->
            <th>MATERIAL</th>
            <th>LABOR</th>
            <th>AMOUNT</th>
            <th>WT.%</th>
            <!-- WORK ACCOMPLISHED columns -->
            <th>PREVIOUS MATERIAL</th>
            <th>PRESENT MATERIAL</th>
            <th>REMAINING MATERIAL</th>
            <th>PREVIOS LABOR</th>
            <th>PRESENT LABOR</th>
            <th>REMAINING LABOR</th>
            <!-- New columns added for table structure -->
            <th>PREVIOUS</th>
            <th>ACTUAL MATERIALS %</th>
            <th>REMAINING</th>
            <th>RESEREVE</th>
          </tr>
        </thead>
        <tbody v-for="section in sections" :key="section.id">
          <tr>
            <td colspan="16" class="font-weight-bold">
              {{ section.letter_label_for_item_no }}
            </td>
          </tr>
          <tr>
            <td colspan="16" class="font-weight-bold">
              {{ section.header_per_project_section }}
            </td>
          </tr>
          <tr>
            <td colspan="16">
              <v-progress-linear
                :value="getProgressPercentage(section)"
                :color="getProgressColor(getSumWtPercent(section), getSectionTotalWt(section))"
                height="20"
                striped
              >
                <template v-slot:default>
                  <strong>{{ formatDecimal(getSumWtPercent(section)) }}/{{ formatDecimal(getSectionTotalWt(section)) }}</strong>
                </template>
              </v-progress-linear>
            </td>
          </tr>
          <tr>
            <td colspan="16" class="font-italic">
              {{ section.mainDescription }}
            </td>
          </tr>
          <tr v-for="item in section.items" :key="'item-' + item.id">
            <td>{{ item.itemno }}</td>
            <td @click="toggleMaterialDetails(section.id, item.itemno)" style="cursor: pointer;">
              {{ item.subDescription }}
            </td>
            <td>{{ formatNumberWithCommas(
                        getMaterialModifieds(section.id, item.itemno).reduce((sum, material, index) => 
                          sum + (getMaterialQuantityFromMaterials(section.id, item.itemno, index) * material.price), 0)
                      ) }}</td>
            <td>{{ formatNumberWithCommas(
                          getProjectItemModified(section.id, item.itemno).project_workers.reduce((sum, worker) => 
                            sum + (worker.ratePerDay * worker.days), 0)
                        ) }}</td>
            <td>{{ formatNumberWithCommas(
                        getMaterialModifieds(section.id, item.itemno).reduce((sum, material, index) => 
                          sum + (getMaterialQuantityFromMaterials(section.id, item.itemno, index) * material.price), 0) +
                        getProjectItemModified(section.id, item.itemno).project_workers.reduce((sum, worker) => 
                          sum + (worker.ratePerDay * worker.days), 0)
                      ) }}</td>
            <td>{{ formatNumberWithCommas(item.wt_percent) }}</td>
            <!-- Summed values -->
            <td>{{ formatNumberWithCommas(getPreviousMaterial(section.id, item.itemno)) }}</td>
            <td>{{ formatNumberWithCommas(getMaterialCost(section.id, item.itemno)) }}</td>
            <td>{{ formatNumberWithCommas(
                        getMaterialModifieds(section.id, item.itemno).reduce((sum, material) => 
                          sum + parseFloat(material.remainingsubtotal || 0), 0)
                      ) }}</td>
            <td>{{ formatNumberWithCommas(item.previousLabor) }}</td>
            <td>{{ formatNumberWithCommas(item.presentLabor) }}</td>
            <td>{{ formatNumberWithCommas(item.remainingLabor) }}</td>
            <!-- New columns added for table structure -->
            <td>hi</td>
            <td>
              {{ formatTruncatedPercentage((item.presentMaterial * 100) / getRemainingMaterialSum(section.id, item.itemno)) }}%
            </td>
            <td></td>
            <td></td>
          </tr>
          <!-- Material and Labor Details for an item -->
          <tr v-for="item in section.items.filter(i => expandedItems[`${section.id}-${i.itemno}`])" :key="'expanded-details-' + item.id">
            <td colspan="16">
              <!-- Material Details Table -->
              <table class="material-table" border="1">
                <thead>
                  <tr>
                    <th>Input</th>
                    <th>Material</th>
                    <th>Unit</th>
                    <th>Quantity</th>
                    <th>Remaining</th>
                    <th>Price</th>
                    <th>Material Cost</th>
                    <th>Remaining</th>
                    <th>Action</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(material, index) in getMaterialModifieds(section.id, item.itemno)" :key="'material-' + index">
                    <td style="width: 100px;">
                      <input type="number" v-model.number="material.inputField" placeholder="Enter value" />
                    </td>
                    <td>{{ material.material }}</td>
                    <td>{{ material.unit }}</td>
                    <td>{{ formatNumberWithCommas(getMaterialQuantityFromMaterials(section.id, item.itemno, index)) }}</td>
                    <td>{{ formatNumberWithCommas(material.remainingquantity) }}</td>
                    <td>{{ formatNumberWithCommas(material.price) }}</td>
                    <td>
                      {{ formatNumberWithCommas(getMaterialQuantityFromMaterials(section.id, item.itemno, index) * material.price) }}
                    </td>
                    <td>{{ formatNumberWithCommas(material.remainingsubtotal) }}</td>
                    <td v-if="index === 0" :rowspan="getMaterialModifieds(section.id, item.itemno).length">
                      <button @click="updateAllMaterials(section.id, item.itemno)">Update All Materials</button>
                    </td>
                  </tr>
                  <tr v-if="getMaterialModifieds(section.id, item.itemno).length === 0">
                    <td colspan="9">No materials available</td>
                  </tr>
                  <tr>
                    <td colspan="6" class="font-weight-bold">Sub-total</td>
                    <td class="font-weight-bold">
                      {{ formatNumberWithCommas(
                        getMaterialModifieds(section.id, item.itemno).reduce((sum, material, index) => 
                          sum + (getMaterialQuantityFromMaterials(section.id, item.itemno, index) * material.price), 0)
                      ) }}
                    </td>
                    <td></td>
                    <td></td>
                  </tr>
                </tbody>
              </table> 
              
              <!-- Workers Table (Labor Details) with Input Field -->
              <div v-if="getProjectItemModified(section.id, item.itemno) &&
                          getProjectItemModified(section.id, item.itemno).project_workers &&
                          getProjectItemModified(section.id, item.itemno).project_workers.length">
                <table class="workers-table" border="1">
                  <thead>
                    <tr>
                      <th>Input</th>
                      <th>Labor Requirements</th>
                      <th>Name</th>
                      <th>Days</th>
                      <th>Remaining</th>
                      <th>Rate Per Day</th>
                      <th>Labor Cost</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="(worker, wIndex) in getProjectItemModified(section.id, item.itemno).project_workers" :key="'worker-' + wIndex">
                      <td style="width: 100px;">
                        <input type="number" v-model.number="worker.inputField" placeholder="Enter value" />
                      </td>
                      <td>{{ worker.laborRequirments }}</td>
                      <td>{{ worker.name }}</td>
                      <td>{{ worker.days }}</td>
                      <td>{{ worker.remaining }}</td>
                      <td>{{ worker.ratePerDay }}</td>
                      <td>{{ formatNumberWithCommas(worker.ratePerDay * worker.days) }}</td>
                    </tr>
                    <!-- Subtotal row for labor cost -->
                    <tr class="font-weight-bold">
                      <td colspan="6">Subtotal Labor Cost</td>
                      <td>
                        {{ formatNumberWithCommas(
                          getProjectItemModified(section.id, item.itemno).project_workers.reduce((sum, worker) => 
                            sum + (worker.ratePerDay * worker.days), 0)
                        ) }}
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';

export default {
  name: 'ProjectDetails',
  data() {
    return {
      sections: [],
      projectItemModifieds: [],
      expandedItems: {},
      projectWorkers: [],
      showProjectWorkers: false
    };
  },
  computed: {
    totalAmount() {
      return this.sections.reduce((sum, section) =>
        sum + section.items.reduce((acc, item) => acc + parseFloat(item.amount || 0), 0), 0);
    },
    totalWtPercent() {
      return this.sections.reduce((sum, section) =>
        sum + section.items.reduce((acc, item) => acc + parseFloat(item.wt_percent || 0), 0), 0);
    },
    totalPreviousAmount() {
      return this.sections.reduce((sum, section) =>
        sum + section.items.reduce((acc, item) => acc + parseFloat(this.calculatePreviousAmount(section.id, item)), 0), 0);
    },
    totalTotalAmount() {
      return this.sections.reduce((sum, section) =>
        sum + section.items.reduce((acc, item) =>
          acc + parseFloat(this.getTotalAmount(section.id, item.itemno)), 0), 0);
    },
    totalPrevPercentage() {
      return this.sections.reduce((sum, section) =>
        sum + section.items.reduce((acc, item) => acc + parseFloat(this.getPreviousPercentage(section.id, item.itemno)), 0), 0);
    },
    totalLaborCost() {
      return this.projectWorkers.reduce((sum, worker) => sum + (worker.ratePerDay * worker.days), 0);
    }
  },
  methods: {
    async fetchData() {
      const documentId = this.$route.params.documentId;
      try {
        const response = await axios.get(
          `http://localhost:1337/api/header-per-project-sections?populate=*&&filters[project][documentId][$eq]=${documentId}`
        );
        if (response.data && response.data.data.length > 0) {
          this.sections = response.data.data;
        }
      } catch (error) {
        console.error('Error fetching data:', error);
      }
    },
    async fetchProjectItemModifieds() {
      try {
        const response = await axios.get("http://localhost:1337/api/project-item-modifieds?populate=*");
        if (response.data && response.data.data) {
          this.projectItemModifieds = response.data.data;
        }
      } catch (error) {
        console.error('Error fetching project item modifieds:', error);
      }
    },
    async fetchProjectWorkers() {
      const documentId = this.$route.params.documentId;
      try {
        const response = await axios.get("http://localhost:1337/api/projects?populate=*");
        if (response.data && response.data.data) {
          const project = response.data.data.find(p => p.documentId === documentId);
          if (project && project.project_workers) {
            this.projectWorkers = project.project_workers;
          }
        }
      } catch (error) {
        console.error('Error fetching project workers:', error);
      }
    },
    updateProjectItemModifiedAmounts() {
      this.sections.forEach(section => {
        section.project_item_modifieds = this.projectItemModifieds.filter(mod =>
          mod.header_per_project_section && mod.header_per_project_section.id === section.id);
      });
    },
    getPreviousQty(sectionId, itemno) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (!section || !section.project_item_modifieds) return 0;
      const modifiedItem = section.project_item_modifieds.find(modItem => modItem.itemno === itemno);
      return modifiedItem ? parseFloat(modifiedItem.P_EnteredQuantity) || 0 : 0;
    },
    getModifiedQuantity(sectionId, itemno) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (!section || !section.project_item_modifieds) return 0;
      const modifiedItem = section.project_item_modifieds.find(modItem => modItem.itemno === itemno);
      return modifiedItem ? parseFloat(modifiedItem.quantity) || 0 : 0;
    },
    getModifiedAmount(sectionId, itemno) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (!section || !section.project_item_modifieds) return 0;
      const modifiedItem = section.project_item_modifieds.find(modItem => modItem.itemno === itemno);
      return modifiedItem ? parseFloat(modifiedItem.amount) || 0 : 0;
    },
    calculatePreviousAmount(sectionId, item) {
      const previousQty = this.getPreviousQty(sectionId, item.itemno);
      const unitCost = parseFloat(item.unitCost) || 0;
      return previousQty * unitCost;
    },
    getPreviousPercentage(sectionId, itemno) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (!section || !section.project_item_modifieds) return 0;
      const modifiedItem = section.project_item_modifieds.find(modItem => modItem.itemno === itemno);
      return modifiedItem ? parseFloat(modifiedItem.p_wt_percent) || 0 : 0;
    },
    getTotalAmount(sectionId, itemno) {
      const materials = this.getMaterialModifieds(sectionId, itemno);
      const materialCost = materials.reduce((sum, material) => sum + (material.quantity * material.price), 0);
      const laborCost = this.projectWorkers.reduce((sum, worker) => sum + (worker.ratePerDay * worker.days), 0);
      return materialCost + laborCost;
    },
    getMaterialModifieds(sectionId, itemno) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (!section || !section.project_item_modifieds) return [];
      const modifiedItem = section.project_item_modifieds.find(modItem => modItem.itemno === itemno);
      if (!modifiedItem || !modifiedItem.material_modifieds) return [];
      // Ensure each material has reactive properties added.
      modifiedItem.material_modifieds.forEach(material => {
        if (material.inputField === undefined) {
          this.$set(material, 'inputField', 0);
        }
        if (material.remainingquantity === undefined) {
          this.$set(material, 'remainingquantity', material.quantity);
        }
        if (material.remainingsubtotal === undefined) {
          this.$set(material, 'remainingsubtotal', material.subtotal);
        }
      });
      return modifiedItem.material_modifieds;
    },
    getMaterialQuantityFromMaterials(sectionId, itemno, index) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (!section || !section.project_item_modifieds) return 0;
      const modifiedItem = section.project_item_modifieds.find(modItem => modItem.itemno === itemno);
      if (!modifiedItem || !modifiedItem.materials) return 0;
      return modifiedItem.materials[index] ? modifiedItem.materials[index].quantity : 0;
    },
    getRemainingMaterialSum(sectionId, itemno) {
      const materials = this.getMaterialModifieds(sectionId, itemno);
      return materials.reduce((sum, material) => sum + parseFloat(material.remainingquantity || 0), 0);
    },
    getMaterialDifference(sectionId, item) {
      const totalRemaining = this.getRemainingMaterialSum(sectionId, item.itemno);
      if (totalRemaining === 0) return 0;
      const actualPercentage = (item.presentMaterial * 100) / totalRemaining;
      return actualPercentage;
    },
    toggleMaterialDetails(sectionId, itemno) {
      const key = `${sectionId}-${itemno}`;
      this.$set(this.expandedItems, key, !this.expandedItems[key]);
    },
    toggleProjectWorkers() {
      this.showProjectWorkers = !this.showProjectWorkers;
    },
    formatDecimal(value) {
      if (value === null || value === undefined || isNaN(value)) return '0.00';
      return parseFloat(value).toFixed(2);
    },
    formatNumber(value) {
      if (value === null || value === undefined || isNaN(value)) return '0.00';
      return parseFloat(value).toFixed(2);
    },
    formatNumberWithCommas(value) {
      if (value === null || value === undefined || isNaN(value)) return '0';
      const formattedValue = parseFloat(value).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 });
      return formattedValue.endsWith('.00') ? formattedValue.slice(0, -3) : formattedValue;
    },
    formatTruncatedPercentage(value) {
      if (value === null || value === undefined || isNaN(value)) return '--';
      const truncated = Math.floor(value);
      return truncated.toFixed(2);
    },
    redirectToViewWeeklyProgressReport() {
      const documentId = this.$route.params.documentId;
      this.$router.push({ name: 'ViewWeeklyProgressReport', params: { documentId } });
    },
    getSumWtPercent(section) {
      if (section.project_item_modifieds && section.project_item_modifieds.length > 0) {
        return section.project_item_modifieds.reduce((sum, modItem) =>
          sum + (parseFloat(modItem.sum_wt_percent) || 0), 0);
      }
      return 0;
    },
    getSectionTotalWt(section) {
      if (section.items && section.items.length > 0) {
        return section.items.reduce((sum, item) =>
          sum + (parseFloat(item.wt_percent) || 0), 0);
      }
      return 0;
    },
    getProgressColor(progress, total) {
      if (progress >= total) return 'green';
      if (progress < total * 0.3) return 'red';
      if (progress < total * 0.7) return 'orange';
      return 'green';
    },
    getProgressPercentage(section) {
      const total = this.getSectionTotalWt(section);
      if (total === 0) return 0;
      return (this.getSumWtPercent(section) / total) * 100;
    },
    downloadPDF() {
      const margin = 20;
      const headerMargin = 30;
      const tableElement = this.$refs.pdfTable;
      html2canvas(tableElement, { useCORS: true }).then(canvas => {
        const imgData = canvas.toDataURL('image/png');
        const pdf = new jsPDF('p', 'pt', 'a4');
        const pdfWidth = pdf.internal.pageSize.getWidth() - 2 * margin;
        const pdfHeight = (canvas.height * pdfWidth) / canvas.width;
        const now = new Date();
        const timestamp = now.toLocaleString();
        pdf.setFontSize(10);
        pdf.text(`Created on: ${timestamp}`, margin, margin + (headerMargin / 2));
        pdf.addImage(imgData, 'PNG', margin, margin + headerMargin, pdfWidth, pdfHeight);
        pdf.save('project-details.pdf');
      });
    },
    getMaterialCost(sectionId, itemno) {
      const materials = this.getMaterialModifieds(sectionId, itemno);
      return materials.reduce((sum, material) => {
        const enteredQuantity = parseFloat(material.entered_quantity) || 0;
        return sum + (enteredQuantity * material.price);
      }, 0);
    },
    getLaborCost() {
      return this.projectWorkers.reduce((sum, worker) => sum + (worker.ratePerDay * worker.days), 0);
    },
    async updateMaterial(material, showAlert = true) {
      let inputVal = parseFloat(material.inputField);
      if (isNaN(inputVal)) {
        inputVal = 0;
      }
      if (inputVal < 0) {
        if (showAlert) alert("Please enter a valid non-negative number.");
        return;
      }
      const newRemainingQuantity = parseFloat(material.remainingquantity) - inputVal;
      const newRemainingSubtotal = parseFloat(material.remainingsubtotal) - (inputVal * parseFloat(material.price));
      
      try {
        const getResModified = await axios.get(
          `http://localhost:1337/api/material-modifieds?populate=*&filters[documentId][$eq]=${material.documentId}`
        );
        if (getResModified.data && getResModified.data.data && getResModified.data.data.length > 0) {
          const recordModified = getResModified.data.data[0];
          await axios.put(
            `http://localhost:1337/api/material-modifieds/${recordModified.documentId}`,
            {
              data: {
                quantity: newRemainingQuantity,
                subtotal: newRemainingSubtotal,
                entered_quantity: inputVal,
                previous_entered: inputVal
              }
            }
          );
        } else {
          if (showAlert) alert("Material-modified record not found.");
        }
        
        material.remainingquantity = newRemainingQuantity;
        material.remainingsubtotal = newRemainingSubtotal;
        if (showAlert) alert("Material updated successfully!");
      } catch (error) {
        console.error("Error updating material:", error);
        if (showAlert) alert("Update failed. Please try again.");
      }
    },
    async updatePreviousMaterial(sectionId, itemno) {
      // Calculate the material cost for the given item.
      const previousMaterialValue = this.getMaterialCost(sectionId, itemno);
      
      // Find the corresponding modified record and update its previous_material.
      const section = this.sections.find(sec => sec.id === sectionId);
      if (section && section.project_item_modifieds) {
        const modifiedItem = section.project_item_modifieds.find(mod => mod.itemno === itemno);
        if (modifiedItem) {
          try {
            await axios.put(
              `http://localhost:1337/api/project-item-modifieds/${modifiedItem.documentId}`, 
              { data: { previous_material: previousMaterialValue } }
            );
            // Update the local record as well.
            modifiedItem.previous_material = previousMaterialValue;
          } catch (error) {
            console.error("Error updating previous material:", error);
          }
        }
      }
    },
    async updateAllMaterials(sectionId, itemno) {
      const materials = this.getMaterialModifieds(sectionId, itemno);
      for (const material of materials) {
        await this.updateMaterial(material, false);
      }
      // After updating materials, update the previous material value.
      await this.updatePreviousMaterial(sectionId, itemno);
      alert("All materials updated successfully!");
    },
    getPreviousMaterial(sectionId, itemno) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (section && section.project_item_modifieds) {
        const modifiedItem = section.project_item_modifieds.find(mod => mod.itemno === itemno);
        return modifiedItem ? modifiedItem.previous_material : 0;
      }
      return 0;
    },
    updateItemMaterialSums() {
      for (const section of this.sections) {
        for (const item of section.items) {
          let sumEntered = 0;
          let sumPrevious = 0;
          const materials = this.getMaterialModifieds(section.id, item.itemno);
          for (const material of materials) {
            axios.get(
              `http://localhost:1337/api/material-modifieds?populate=*&filters[documentId][$eq]=${material.documentId}`
            )
            .then(response => {
              if (response.data && response.data.data) {
                response.data.data.forEach(record => {
                  sumEntered += Number(record.entered_quantity) || 0;
                  sumPrevious += Number(record.previous_entered) || 0;
                });
              }
            })
            .catch(error => {
              console.error("Error fetching material sums for documentId", material.documentId, error);
            });
          }
          this.$set(item, 'presentMaterial', sumEntered);
          this.$set(item, 'previousMaterial', sumPrevious);
          this.updateRemainingPercentageForItem(section, item);
        }
      }
    },
    updateRemainingPercentageForItem(section, item) {
      const actualPercentage = (item.presentMaterial * 100) / this.getRemainingMaterialSum(section.id, item.itemno);
      const difference = this.getMaterialDifference(section.id, item) - actualPercentage;
      const computedPercentage = this.formatTruncatedPercentage(difference);
      
      axios.put(`http://localhost:1337/api/project-items/${item.documentId}`, {
        data: {
          remaining_percentage: computedPercentage
        }
      })
      .then(response => {
        console.log('Project item updated with remaining_percentage:', response);
      })
      .catch(error => {
        console.error('Error updating project item:', error);
      });
      
      const modifiedRecord = section.project_item_modifieds.find(
        modItem => modItem.itemno === item.itemno
      );
      if (modifiedRecord && modifiedRecord.id) {
        axios.put(`http://localhost:1337/api/project-item-modifieds/${modifiedRecord.documentId}`, {
          data: {
            remaining_percentage: computedPercentage
          }
        })
        .then(response => {
          console.log('Project item modified updated with remaining_percentage:', response);
        })
        .catch(error => {
          console.error('Error updating project item modified:', error);
        });
      } else {
        console.warn('No matching modified record found for item:', item.itemno);
      }
    },
    updateWeightPercent() {
      let globalMaterialCount = 0;
      this.sections.forEach(section => {
        section.items.forEach(item => {
          const localMaterialCount = this.getMaterialModifieds(section.id, item.itemno).length;
          globalMaterialCount += localMaterialCount;
        });
      });
      if (globalMaterialCount === 0) return;
      const factor = 100 / globalMaterialCount;
      this.sections.forEach(section => {
        section.items.forEach(item => {
          const localMaterialCount = this.getMaterialModifieds(section.id, item.itemno).length;
          const newWtPercent = factor * localMaterialCount;
          this.$set(item, 'wt_percent', newWtPercent);
        });
      });
    },
    // Helper method to get the modified record for a given section and item.
    getProjectItemModified(sectionId, itemno) {
      const section = this.sections.find(sec => sec.id === sectionId);
      if (!section || !section.project_item_modifieds) return null;
      return section.project_item_modifieds.find(modItem => modItem.itemno === itemno);
    }
  },
  mounted() {
    Promise.all([
      this.fetchData(),
      this.fetchProjectItemModifieds(),
      this.fetchProjectWorkers()
    ])
      .then(() => {
        this.updateProjectItemModifiedAmounts();
        this.updateItemMaterialSums();
        this.updateWeightPercent();
      })
      .catch(error => {
        console.error('Error in mounted hook:', error);
      });
  }
};
</script>

<style scoped>
.table-container {
  width: 100%;
  margin: 0 auto;
  padding: 0 10px;
  overflow-x: auto;
}

table {
  width: 100%;
  background-color: #ffffff;
  border-collapse: collapse;
  table-layout: fixed;
}

th,
td {
  padding: 4px;
  text-align: center;
  font-size: 12px;
  word-wrap: break-word;
}

th {
  background-color: #f4f4f4;
}

.table-header {
  background-color: rgb(239, 213, 40);
  font-size: 16px;
  font-weight: bold;
  text-align: center;
  padding: 8px;
}

.project-info {
  font-weight: normal;
  font-size: 10px;
  padding: 6px;
}

.font-weight-bold {
  font-weight: bold;
  background-color: #e0e0e0;
}

.font-italic {
  font-style: italic;
}

.bg-light {
  background-color: #f8f9fa;
}

.redirect-btn {
  background: #012b86;
  margin-left: 50px;
  margin-top: 8px;
  padding: 8px;
  border-radius: 5px;
  color: white;
  font-size: 12px;
}

.download-btn {
  background: #ff0000;
  margin-left: 8px;
  margin-top: 8px;
  padding: 8px;
  border-radius: 5px;
  color: white;
  font-size: 12px;
}

.pdf-table {
  border: 1px solid #ddd;
}

.pdf-table th,
.pdf-table td {
  border: 1px solid #ddd;
}

.header-image {
  width: 100%;
  height: 140px;
  padding-left: 10px;
  padding-right: 10px;
  background-color: #ffffff;
}

.material-table {
  width: 100%;
  margin-top: 5px;
  border-collapse: collapse;
}

.material-table th,
.material-table td {
  padding: 4px;
  border: 1px solid #ccc;
  font-size: 11px;
}

.workers-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 10px;
}

.workers-table th,
.workers-table td {
  padding: 6px;
  border: 1px solid #ddd;
  font-size: 12px;
  text-align: center;
}
</style>
