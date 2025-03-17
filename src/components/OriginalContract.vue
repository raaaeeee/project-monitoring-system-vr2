<template>
  <div class="form-wrapper mt-3 mx-15">
    <!-- First Form: Project Section -->
    <div class="form-container">
      <h2>New Project Section</h2>
      <form @submit.prevent="submitSection">
        <div class="form-group">
          <label for="letterLabel">Letter Label</label>
          <input v-model="formData.letter_label_for_item_no" id="letterLabel" type="text" required @input="disableProjectItemButton" />
        </div>

        <div class="form-group">
          <label for="mainDescription">Main Description</label>
          <input v-model="formData.mainDescription" id="mainDescription" type="text" required @input="disableProjectItemButton" />
        </div>

        <button type="submit" class="btn mt-5" :disabled="isSubmitting">Submit Section</button>
      </form>
    </div>

    <!-- Second Form: Project Items -->
    <div class="form-container">
      <h2>Project Item Details</h2>
      <form @submit.prevent="submitProjectItem">
        <div v-if="!sectionId" class="alert">
          <p><strong>Note:</strong> Please submit the Project Section first.</p>
        </div>

        <div class="form-fields">
          <div class="form-group">
            <label for="itemno">Item Number</label>
            <input v-model="projectItem.itemno" id="itemno" type="text" required :disabled="!sectionId" />
          </div>

          <div class="form-group">
            <label for="subDescription">Sub Description</label>
            <input v-model="projectItem.subDescription" id="subDescription" type="text" required :disabled="!sectionId" />
          </div>
        </div>

        <button type="submit" class="btn mt-5" id="orange" :disabled="!sectionId || isSubmitting || isProjectItemButtonDisabled" @focus="enableProjectItemButton">Submit Item</button>
        <button type="button" class="btn mt-5" @click="showLaborForm = true" :disabled="!sectionId">Add Labor Requirements</button>
      </form>
    </div>

    <!-- Third Form: Material Details -->
    <div v-if="showMaterialForm" class="modal">
      <div class="modal-content">
        <h2>Material Details</h2>
        <form @submit.prevent="submitMaterial">
          <div class="form-group">
            <label for="material">Material</label>
            <input v-model="materialData.material" id="material" type="text" required />
          </div>

          <div class="form-group">
            <label for="materialQuantity">Quantity</label>
            <input v-model.number="materialData.quantity" id="materialQuantity" type="number" step="0.01" required />
          </div>

          <div class="form-group">
            <label for="materialUnit">Unit</label>
            <input v-model="materialData.unit" id="materialUnit" type="text" required />
          </div>

          <div class="form-group">
            <label for="materialPrice">Price</label>
            <input v-model.number="materialData.price" id="materialPrice" type="number" step="0.01" required />
          </div>

          <button type="submit" class="btn mt-5" :disabled="isSubmitting">Submit Material</button>
          <button type="button" class="btn mt-5" @click="done" :disabled="isSubmitting">Done</button>
        </form>
      </div>
    </div>

    <!-- Labor Requirements Form -->
    <div v-if="showLaborForm" class="modal">
      <div class="modal-content">
        <h2>Labor Requirements</h2>
        <form @submit.prevent="submitLabor">
          <div class="form-group">
            <label for="laborRequirements">Labor Requirements</label>
            <input v-model="laborData.laborRequirements" id="laborRequirements" type="text" required />
          </div>

          <div class="form-group">
            <label for="name">Name</label>
            <input v-model="newPerson" id="name" type="text" @keyup.enter="addPerson" />
            <button type="button" @click="addPerson">Add Person</button>
            <div class="scrollable-list">
              <ul>
                <li v-for="(person, index) in laborData.name" :key="index">
                  {{ person }} <button type="button" @click="removePerson(index)">Remove</button>
                </li>
              </ul>
            </div>
          </div>

          <div class="form-fields">
            <div class="form-group">
              <label for="manpower">Manpower</label>
              <input v-model.number="laborData.manpower" id="manpower" type="number" required />
            </div>

            <div class="form-group">
              <label for="days">Days</label>
              <input v-model.number="laborData.days" id="days" type="number" required />
            </div>

            <div class="form-group">
              <label for="ratePerDay">Rate/Day</label>
              <input v-model.number="laborData.ratePerDay" id="ratePerDay" type="number" step="0.01" required />
            </div>
          </div>

          <button type="submit" class="btn mt-5" :disabled="isSubmitting">Submit Labor</button>
          <button type="button" class="btn mt-5" @click="showLaborForm = false" :disabled="isSubmitting">Cancel</button>
        </form>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  name: "OriginalContract",
  data() {
    return {
      formData: {
        letter_label_for_item_no: "",
        mainDescription: "",
        project: null,
        project_with_modified_datum: null,
      },
      projectItem: {
        itemno: "",
        subDescription: "",
        quantity: null,
        unit: "",
        unitCost: null,
        amount: null,
        wt_percent: null,
        header_per_project_section: null,
      },
      materialData: {
        material: "",
        quantity: null,
        unit: "",
        price: null,
      },
      laborData: {
        laborRequirements: "",
        name: [],
        manpower: null,
        days: null,
        ratePerDay: null,
      },
      newPerson: "",
      sectionId: null,
      projectItemId: null,
      projectItemModifiedId: null,
      isSubmitting: false,
      isProjectItemButtonDisabled: false,
      showMaterialForm: false,
      showLaborForm: false,
    };
  },
  created() {
    // Extract projectId and secondprojectId from the URL
    this.formData.project = Number(this.$route.query.projectId) || null;
    this.formData.project_with_modified_datum = Number(this.$route.query.secondprojectId) || null;

    console.log("Extracted Project ID:", this.formData.project);
    console.log("Extracted Second Project ID:", this.formData.project_with_modified_datum);

    // Redirect to the home page if no project ID is found
    if (!this.formData.project || !this.formData.project_with_modified_datum) {
      alert("No project ID found.");
      this.$router.push("/");
    }
  },
  methods: {
    disableProjectItemButton() {
      this.isProjectItemButtonDisabled = true;
    },
    enableProjectItemButton() {
      this.isProjectItemButtonDisabled = false;
    },
    async submitSection() {
      if (this.isSubmitting) return;
      this.isSubmitting = true;
      try {
        console.log("Submitting section with:", this.formData);

        const sectionResponse = await axios.post(
          "http://localhost:1337/api/header-per-project-sections?populate=*",
          {
            data: this.formData,
          }
        );

        console.log("Section created:", sectionResponse.data);
        this.sectionId = sectionResponse.data.data.id;
        this.projectItem.header_per_project_section = this.sectionId;

        this.isProjectItemButtonDisabled = false;
        alert("Project Section submitted successfully!");
      } catch (error) {
        console.error("Error submitting section:", error.response?.data || error);
        alert("Failed to submit project section.");
      } finally {
        this.isSubmitting = false;
      }
    },

    resetProjectItem() {
      this.projectItem = {
        itemno: "",
        subDescription: "",
        quantity: null,
        unit: "",
        unitCost: null,
        amount: null,
        wt_percent: null,
        header_per_project_section: this.sectionId,
      };
    },

    async submitProjectItem() {
      if (this.isSubmitting) return;
      this.isSubmitting = true;
      try {
        if (!this.sectionId) {
          alert("Please submit the Project Section first.");
          return;
        }

        const itemDataForFirstAPI = { data: this.projectItem };
        const itemDataForSecondAPI = {
          data: {
            ...this.projectItem,
            header_per_project_section: this.sectionId,
          },
        };

        // Remove wt_percent for the second API
        delete itemDataForSecondAPI.data.wt_percent;

        // Submit the project item and store its ID
        const response = await axios.post("http://localhost:1337/api/project-items", itemDataForFirstAPI);
        this.projectItemId = response.data.data.id;

        // Submit to project-item-modifieds and capture the modified item ID for later use
        const modifiedResponse = await axios.post("http://localhost:1337/api/project-item-modifieds", itemDataForSecondAPI);
        this.projectItemModifiedId = modifiedResponse.data.data.id;

        alert("Project Item submitted successfully!");
        this.resetProjectItem();
        this.showMaterialForm = true;
      } catch (error) {
        console.error("Error submitting project item:", error.response?.data || error);
        alert("Failed to submit project item.");
      } finally {
        this.isSubmitting = false;
      }
    },

    async submitMaterial() {
      if (this.isSubmitting) return;
      this.isSubmitting = true;
      try {
        console.log("Submitting material with:", this.materialData);

        // Ensure that the project item modified ID is available for the relation
        if (!this.projectItemModifiedId) {
          alert("No project item modified found. Please submit a project item first.");
          return;
        }

        // Prepare the payload for the /materials endpoint using the correct relation field and computing subtotal
        const materialDataPayload = {
          ...this.materialData,
          project_item_modified: this.projectItemModifiedId,
          subtotal: this.materialData.quantity * this.materialData.price,
        };

        // Submit material to the original materials API with the corrected payload
        const materialResponse = await axios.post(
          "http://localhost:1337/api/materials",
          {
            data: materialDataPayload,
          }
        );
        console.log("Material created:", materialResponse.data);

        // Prepare payload for the material-modifieds API (unchanged as it already uses the correct relation field)
        const materialModifiedPayload = {
          data: {
            material: this.materialData.material,
            quantity: this.materialData.quantity,
            unit: this.materialData.unit,
            price: this.materialData.price,
            project_item_modified: this.projectItemModifiedId,
            subtotal: this.materialData.quantity * this.materialData.price,
          },
        };

        // Submit to the material-modifieds API
        const materialModifiedResponse = await axios.post(
          "http://localhost:1337/api/material-modifieds",
          materialModifiedPayload
        );
        console.log("Material modified created:", materialModifiedResponse.data);

        alert("Material submitted successfully!");
        this.resetMaterialData();
      } catch (error) {
        console.error("Error submitting material:", error.response?.data || error);
        alert("Failed to submit material.");
      } finally {
        this.isSubmitting = false;
      }
    },
    resetMaterialData() {
      this.materialData = {
        material: "",
        quantity: null,
        unit: "",
        price: null,
      };
    },

    async submitLabor() {
      if (this.isSubmitting) return;
      this.isSubmitting = true;
      try {
        console.log("Submitting labor with:", this.laborData);

        if (!this.projectItemModifiedId) {
          alert("No project item modified found. Please submit a project item first.");
          return;
        }

        // Construct the payload using the relation field "project_item_modified"
        const laborPayload = {
          laborRequirments: this.laborData.laborRequirements,
          manpower: this.laborData.manpower,
          days: this.laborData.days,
          ratePerDay: this.laborData.ratePerDay,
          project_item_modified: this.projectItemModifiedId,
          name: this.laborData.name.join(", ")
        };

        // Submit labor data to the project-workers API
        const laborResponse = await axios.post(
          "http://localhost:1337/api/project-workers",
          { data: laborPayload }
        );
        console.log("Labor submitted:", laborResponse.data);

        alert("Labor submitted successfully!");
        this.resetLaborData();
        this.showLaborForm = false;
      } catch (error) {
        console.error("Error submitting labor:", error.response?.data || error);
        alert("Failed to submit labor.");
      } finally {
        this.isSubmitting = false;
      }
    },
    resetLaborData() {
      this.laborData = {
        laborRequirements: "",
        name: [],
        manpower: null,
        days: null,
        ratePerDay: null,
      };
    },
    addPerson() {
      if (this.newPerson.trim() !== "") {
        this.laborData.name.push(this.newPerson.trim());
        this.newPerson = "";
      }
    },
    removePerson(index) {
      this.laborData.name.splice(index, 1);
    },
    done() {
      this.showMaterialForm = false;
      this.showLaborForm = true;
      alert("Material submission complete! Please fill out the Labor Requirements form.");
    },
  },
};
</script>

<style scoped>
.form-wrapper {
  display: flex;
  flex-direction: row;
  justify-content: space-evenly;
  flex-wrap: wrap;
  background-color: white;
  border-radius: 10px;
}
.form-container {
  max-width: 500px;
  width: 100%;
  padding: 20px;
  border-radius: 10px;
  flex-grow: 1;
}
h2 {
  text-align: center;
  margin-bottom: 20px;
}
.form-fields {
  display: flex;
  justify-content: space-between;
  gap: 10px;
}
.form-group {
  margin-bottom: 10px;
}
label {
  display: block;
  font-weight: bold;
  margin-bottom: 5px;
}
input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
  font-size: 16px;
}
input:disabled {
  background: #f5f5f5;
  cursor: not-allowed;
}
.btn {
  width: 100%;
  padding: 10px;
  background: #066913;
  color: white;
  border: none;
  border-radius: 5px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.3s;
}
.btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}
.alert {
  background: #fff3cd;
  padding: 5px;
  border: 1px solid #ffeeba;
  color: #856404;
  margin-bottom: 5px;
  border-radius: 5px;
}

#orange {
  background: rgb(255, 179, 1);
  color: black;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0, 0, 0, 0.5);
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 10px;
  max-width: 500px;
  width: 100%;
}
.scrollable-list {
  max-height: 100px;
  overflow-y: auto;
  margin-top: 10px;
}
.scrollable-list ul {
  list-style-type: none;
  padding: 0;
}
.scrollable-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 5px;
}
</style>
