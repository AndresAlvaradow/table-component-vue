<template>
    <div class="q-px-md q-pb-sm">
        <!-- formulario para ediatr elvalor de un campo  -->
        <q-dialog v-model="editDialog" persistent>
            <q-card>
                <q-card-section class="row items-center q-gutter-sm">
                    <!-- Establece el tipo como number y agrega un patrón para limitar a 2 dígitos -->
                    <!-- <q-input v-model="editValue" :label="labelSelectedField" autofocus type="number" :maxlength="2" :pattern="\d{1,2}" /> -->
                    <q-input
                        v-model="editValue"
                        stack-label
                        filled
                        :label="labelSelectedField"
                        :dense="true"
                        autocomplete="no-suggestions"
                        autofocus
                        maxlength="2"
                        lazy-rules
                        :rules="[validateByTwoCaracterNames]"
                        mask="##"
                    />
                    <!-- <div class="col-12 q-pa-sm">
                            <q-input stack-label filled v-model="editValue" label="labelSelectedField" lazy-rules :rules="[validateCaracter]"
                                :dense="true" autocomplete="no-suggestions" maxlength="2" />
                        </div> -->
                </q-card-section>

                <q-card-section align="right">
                    <q-btn flat label="Cancelar" color="primary" v-close-popup />
                    <q-btn flat label="Editar" color="primary" @click="saveEdit" />
                </q-card-section>
            </q-card>
        </q-dialog>

        <q-table
            :rows="auxOriginalRows"
            :columns="dataTable.columns"
            :row-key="rowKey"
            :no-data-label="labelNoData"
            :rows-per-page-label="rowsPerLabel"
            v-model:pagination="pagination"
            :rows-per-page-options="perPageOptions"
            :filter="filter"
            :title="title"
            :style="{ height: heightTable + '!important' }"
            virtual-scroll
            @request="emitEvent"
            :loading="dataTable.loading"
            @row-dblclick="handleRowDblClick"
            class="my-sticky-header-table-general"
            dense
            :hide-bottom="hideBottomTable"
            :selected-rows-label="getSelectedString"
            :selection="typeSelection"
            v-model:selected="selected"
            @update:selected="updateSelection"
        >
            <!-- slot para el loading recibe un booleano
            este valor se encuentra en la información de la tabla -->
            <template v-slot:loading>
                <q-inner-loading showing color="primary" />
            </template>

            <!-- la seleccion de las columnas con los check -->

            <template v-slot:body-selection="scope" v-if="activateSelector">
                <q-checkbox v-model="scope.selected" class="my-checkbox-container" />
            </template>

            <!-- me permite poder truncar los textos que ingresan a la tabla
            y si no trae valor le agrega Sin registro -->

            <template v-slot:body-cell="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <div>
                        <span v-if="props.value && props.value.length > textSize">
                            {{ truncateContent(props.value) }}
                            <q-tooltip>{{ props.row[props.col.field] }}</q-tooltip>
                        </span>
                        <span v-else>
                            <span v-if="props.value == '' || !props.value"> Sin registro </span>
                            <span v-else>
                                {{ props.value }}
                            </span>
                        </span>
                    </div>
                </q-td>
            </template>

            <!-- slot para el filtro de la tabla props que se utiliza es el de 
            filterTable que es un bool y por defecto esta en true ademas recibe
            toolTipsSearch que es un string y es el toolTips de la busqueda -->

            <template v-if="filterTable" v-slot:top-right>
                <q-tooltip>{{ toolTipsSearch }}</q-tooltip>
                <q-input borderless dense debounce="300" v-model="filter" placeholder="Buscar">
                    <template v-slot:append>
                        <q-icon name="search" />
                    </template>
                </q-input>
            </template>

            <!-- slot para mostrar la numeracion de la tabla
            la data que requiere es en los rows tener el atributo index
            el que se calcula con un helper que tiene la funcion formatRowsTable-->

            <template v-slot:body-cell-index="props">
                <!-- Agrega aquí la lógica para mostrar el índice -->
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <span>{{ props.row.index }}</span>
                </q-td>
            </template>

            <!-- slot de acciones para activarlo se debe colocar el nombre
            actions en el name al declarar la columna 
            props que usa son:
            btnEdit un bool y permite ocultar o mostrar el icono de editar, por defrecto en true
            btnStatus un bool permite cambir los estados de los registros,  por defecto true 
            btnOpcion un bool son los tres puntos pero solo permite una accion por defecto true  -->

            <template v-slot:body-cell-actions="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-btn
                        v-if="btnEdit"
                        dense
                        round
                        flat
                        color="primary"
                        icon="edit"
                        @click="showFormGeneralModal(props.row)"
                    >
                        <q-tooltip>Editar {{ labelEdit }}</q-tooltip>
                    </q-btn>
                    <q-btn
                        v-if="btnStatus"
                        dense
                        round
                        flat
                        :color="props.row.status != 'Inactivo' ? 'negative' : 'positive'"
                        :icon="props.row.status != 'Inactivo' ? 'delete' : 'done'"
                        @click="showConfirmStateModal(props.row)"
                    >
                        <q-tooltip v-if="props.row.status == 'Activo'"> Desactivar {{ labelStatus }} </q-tooltip>
                        <q-tooltip v-else> Activar {{ labelStatus }} </q-tooltip>
                    </q-btn>
                    <q-btn-dropdown
                        v-if="btnOpcion"
                        class="btn-more-opt"
                        dense
                        round
                        flat
                        color="grey"
                        icon="more_vert"
                    >
                        <template v-slot:label>
                            <q-tooltip>
                                {{ toolTipsOpcion }}
                            </q-tooltip>
                        </template>
                        <q-list>
                            <q-item tag="label" v-ripple @click="redirectAction(props.row)">
                                <q-item-section>
                                    <q-item-label>{{ labelOpcion }}</q-item-label>
                                </q-item-section>
                            </q-item>
                        </q-list>
                    </q-btn-dropdown>
                </q-td>
            </template>

            <!-- slot actionOpcion para usar se debe agregar actionsOpcion en el name 
            al declarar la column los props:
            toolTipsOpcion es un string,
            dataOption: es un objeto el cual me permite enviar las acciones
            ejemplo del objeto:
            const dataAccionOpcion = ref([
            { icon: true, iconName: 'contact_support', labelOpcion: 'Atender Contactos', 
            function: (data) => attendContactUs(data) },])
            el icono si se va en false no es necesario enviar el iconName 
            btnAccionGeneral es un btn para cualquier accion esta por defecto en false para activar
            se debe btnAccionGeneral colcoar en true y enviar la funcion en btnAccionGeneralFuction
            su tool tips se agrega en labelAccionGeneral todos estos se pasan por props desde el padre
            se puede enviar un color de icono en colorAccionGeneral y el ico en iconAccionGeneral
            estos son personalizados -->

            <template v-slot:body-cell-actionsOpcion="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-btn
                        v-if="btnAccionGeneral"
                        dense
                        round
                        flat
                        :color="colorAccionGeneral"
                        :icon="iconAccionGeneral"
                        @click="btnAccionGeneralFuction(props.row)"
                    >
                        <q-tooltip> {{ labelAccionGeneral }}</q-tooltip>
                    </q-btn>

                    <q-btn v-if="btnEdit" dense round flat color="primary" icon="edit" @click="editfuction(props.row)">
                        <q-tooltip>Editar {{ labelEdit }}</q-tooltip>
                    </q-btn>
                    <q-btn
                        v-if="btnStatus"
                        dense
                        round
                        flat
                        :color="props.row.colorIconStatus"
                        :icon="props.row.iconStatus"
                        @click="editStateFuction(props.row)"
                    >
                        <q-tooltip v-if="props.row.iconStatus == 'done'"> Desactivar {{ labelStatus }} </q-tooltip>
                        <q-tooltip v-else> Activar {{ labelStatus }} {{ props.row.labelStateGeneral }} </q-tooltip>
                    </q-btn>
                    <q-btn-dropdown
                        v-if="btnActionsOpcion"
                        class="btn-more-opt"
                        dense
                        round
                        flat
                        color="grey"
                        icon="more_vert"
                    >
                        <template v-slot:label>
                            <q-tooltip>
                                {{ toolTipsOpcion }}
                            </q-tooltip>
                        </template>
                        <q-list v-for="data in dataOption" :key="data.labelOpcion">
                            <q-item tag="label" v-ripple @click="data.function(props.row)">
                                <q-item-section>
                                    <q-item-label>
                                        <q-icon v-if="data.icon" :name="data.iconName" />
                                        {{ data.labelOpcion }}</q-item-label
                                    >
                                </q-item-section>
                            </q-item>
                        </q-list>
                    </q-btn-dropdown>
                </q-td>
            </template>

            <!-- slot para el status solo fuciona para Activo o Inactivo
            el name de la columan es status
            no requiere props -->

            <template #body-cell-status="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-chip
                        :color="props.row.status === 'Activo' ? 'green' : 'red'"
                        text-color="white"
                        dense
                        class="text-weight-bolder"
                        square
                        >{{ props.row.status }}</q-chip
                    >
                </q-td>
            </template>

            <!-- Slot para usar en edicion devalores de un campo de la fila  -->
            <template v-slot:body-cell-labelStatusForm="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="openEditDialog(props.row, props.col.field)"
                >
                    {{ props.value }}
                </q-td>
            </template>
            <!-- slot labelStateGeneral sirve para agregar diferentes estados
            se debe agregar la label y el color para que funcione eso se obtiene
            al formatear el row con el helper format-functions
            props que recibe son labelStatePopUp por defecto en false y si se cambia a true
            va a permitir que si hago click sobre el estado realice una accion -->

            <template v-slot:body-cell-labelStateGeneral="props">
                <q-td
                    :props="props"
                    v-if="!labelStatePopUp"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-chip
                        :color="props.row.colorState"
                        text-color="white"
                        dense
                        class="text-weight-bolder"
                        square
                        @click="functionChipsStateGeneral(props.row)"
                        clickable
                        >{{ props.row.labelStateGeneral }}</q-chip
                    >
                </q-td>
                <q-td
                    :props="props"
                    v-else
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-chip :color="props.row.colorState" text-color="blue" dense class="text-weight-bolder" square>{{
                        props.row.labelStateGeneral
                    }}</q-chip>
                </q-td>
            </template>
            <!-- slot priorityGeneral sirve para agregar las prioridades 
            se debe agregar la label y el color para que funcione eso se obtiene
            al formatear el row con el helper format-functions
            no recibe props -->

            <template v-slot:body-cell-priorityGeneral="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-chip
                        :color="props.row.colorPriority"
                        text-color="white"
                        dense
                        class="text-weight-bolder"
                        square
                        >{{ props.row.labelPriority }}</q-chip
                    >
                </q-td>
            </template>

            <!-- slot priorityGeneral sirve para agregar los tipos 
            se debe agregar la label y el color para que funcione eso se obtiene
            al formatear el row con el helper format-functions
            no recibe props -->

            <template v-slot:body-cell-TypeGeneral="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-chip :color="props.row.colorType" text-color="white" dense class="text-weight-bolder" square>{{
                        props.row.labelType
                    }}</q-chip>
                </q-td>
            </template>
            <!-- slot RoleGeneral sirve para agregar los roles 
            se debe agregar la label y el color para que funcione eso se obtiene
            al formatear el row con el helper format-functions
            no recibe props -->

            <template v-slot:body-cell-RoleGeneral="props">
                <q-td
                    :props="props"
                    :class="[{ 'selected-row': props.row.selected }]"
                    @click="toggleRowSelection(props.row)"
                >
                    <q-chip :color="props.row.colorRole" text-color="white" dense class="text-weight-bolder" square>{{
                        props.row.labelRole
                    }}</q-chip>
                </q-td>
            </template>

            <!-- <template v-slot:body-row="props">

                <q-tr :props="props.row" class="cursor-pointer" @click="selectRow(props.row)"
                    :class="{ 'selected-row': props.row.index === selectedRow }"
                    @dblclick="handleDoubleClick(props.row)">
                </q-tr>
            </template>

            <template v-slot:body-selection="props">

                <q-tr :props="props.row" class="cursor-pointer" @click="selectRow(props.row)"
                    :class="{ 'selected-row': props.row.index === selectedRow }"
                    @dblclick="handleDoubleClick(props.row)">
                </q-tr>

            </template> -->

            <template v-slot:header="props">
                <q-tr>
                    <!-- seleccion de las filas -->
                    <q-th v-if="activateSelector"><q-checkbox v-model="props.selected" class="text-center" /></q-th>
                    <!-- fin seleccin toda las filas -->
                    <q-th
                        v-for="col in dataTable.columns"
                        :key="col.name"
                        :align="col.align"
                        @contextmenu.prevent="handleHeaderClick(col, props)"
                    >
                        <template v-if="col.columnPopUpFilter">
                            <q-popup-proxy>
                                <q-banner>
                                    <!-- <template v-slot:avatar>
                                        <q-icon name="signal_wifi_off" color="primary" />

                                    </template> -->
                                    <q-input
                                        :model-value="inputSearchColumn"
                                        label="Búsqueda...."
                                        dense
                                        filled
                                        v-if="filterColumns"
                                        type="search"
                                        @update:model-value="searchColumnFilter($event, col.field)"
                                    >
                                        <template v-slot:append>
                                            <q-icon name="search" />
                                        </template>
                                    </q-input>

                                    <div class="horizontal-checkbox-container" v-if="filterCheckColumns">
                                        <q-checkbox
                                            v-for="value in getUniqueValues(col.field)"
                                            :key="value"
                                            :val="value"
                                            v-model="arraycolumnFilter"
                                            :label="value"
                                            color="primary"
                                            keep-color
                                            @click="applyFilters(col.field)"
                                        />
                                    </div>
                                </q-banner>
                                <div></div>
                            </q-popup-proxy>
                        </template>
                        {{ col.label }}
                        <q-checkbox
                            v-if="col.sortable"
                            v-model="right2"
                            checked-icon="arrow_upward"
                            unchecked-icon="arrow_downward"
                            :color="getColumnColor(col.field)"
                            keep-color
                            @click="sortColumn(right2, col.field)"
                        />
                    </q-th>
                </q-tr>
            </template>

            <!-- loadin de la paginacion se por defecto esta en false y si se quiere utilizar se debe enviar
            un booleano por loadPagination el cual permite mostrar y ocultar el snipper -->

            <template v-if="loadPagination" v-slot:bottom>
                <div class="loadingPaginate">
                    <!-- <q-spinner-hourglass color="primary" size="2em" /> -->
                    <q-spinner color="primary" size="3em" />
                </div>
            </template>
        </q-table>
        <ConfirmModalComponent
            v-if="show_confirm_modal"
            v-model="show_confirm_modal"
            v-bind:message="message"
            @update:show_confirm_modal="closeDialog"
            @update:result="resultDialog"
        />
    </div>
</template>

<script>
import { inject, ref, reactive, toRefs, onMounted, watch } from "vue";
import { TypeRol } from "@/helpers/global/type/type_rol";
import ConfirmModalComponent from "@/ui/components/ConfirmModalComponent.vue";
import { validateByTwoCaracterNames } from "@/helpers/validators/inputRules.ts";
import { useQuasar } from "quasar";
export default {
    name: "TablaGeneral",
    components: {
        ConfirmModalComponent,
    },
    props: {
        labelNoData: {
            type: String,
            default: "",
        },
        title: {
            type: String,
            default: "Tabla",
        },
        rowsPerLabel: {
            type: String,
            default: "Registros",
        },
        perPageOptions: {
            type: Array,
            default: () => [],
        },
        rowKey: {
            type: String,
            default: "field",
        },
        filterTable: {
            type: Boolean,
            default: true,
        },
        // prop para pasar funcione de edicion de campos de una fila
        updateFieldValueFunction: {
            type: Function,
            required: false,
            default: () => {
                // Tu función por defecto aquí
                // Puedes devolver una función vacía o cualquier otra lógica por defecto
            },
        },
        dataUserLogin: {
            type: Object,
            required: false,
            default: () => ({}),
        },
        editfuction: {
            type: Function,
            required: false,
            default: () => {
                // Tu función por defecto aquí
                // Puedes devolver una función vacía o cualquier otra lógica por defecto
            },
        },
        editStateFuction: {
            type: Function,
            required: false,
            default: () => {
                // Tu función por defecto aquí
                // Puedes devolver una función vacía o cualquier otra lógica por defecto
            },
        },
        editProfileFuction: {
            type: Function,
            required: false,
            default: () => {
                // Tu función por defecto aquí
                // Puedes devolver una función vacía o cualquier otra lógica por defecto
            },
        },
        btnAccionGeneralFuction: {
            type: Function,
            required: false,
            default: () => {
                // Tu función por defecto aquí
                // Puedes devolver una función vacía o cualquier otra lógica por defecto
            },
        },
        btnEdit: {
            type: Boolean,
            default: true,
        },
        labelEdit: {
            type: String,
            default: "Usuario",
        },
        btnAccionGeneral: {
            type: Boolean,
            default: false,
        },
        labelAccionGeneral: {
            type: String,
            default: "tooltops del boton",
        },
        colorAccionGeneral: {
            type: String,
            default: "primary",
        },
        iconAccionGeneral: {
            type: String,
            default: "visibility",
        },
        btnStatus: {
            type: Boolean,
            default: true,
        },
        btnActionsOpcion: {
            type: Boolean,
            default: true,
        },
        labelStatus: {
            type: String,
            default: "Usuario",
        },
        btnOpcion: {
            type: Boolean,
            default: true,
        },
        toolTipsOpcion: {
            type: String,
            default: "Opciones de usuario",
        },
        labelOpcion: {
            type: String,
            default: "Asignar perfil",
        },
        dataOption: {
            type: Object,
            required: false, // Agrega esta línea si deseas que user sea requerido
            default: () => ({}), // Puedes agregar un valor por defecto si es apropiado
        },
        labelStatePopUp: {
            type: Boolean,
            default: false,
        },
        toolTipsSearch: {
            type: String,
            default: "Opciones de búsqueda",
        },
        dataSearchColums: {
            type: Object,
            required: false,
            default: () => ({}),
        },
        loadPagination: {
            type: Boolean,
            default: false,
        },
        dataTable: {
            type: Object,
            required: false, // Agrega esta línea si deseas que user sea requerido
            default: () => ({}), // Puedes agregar un valor por defecto si es apropiado
        },
        handleRowDblClick: {
            type: Function,
            required: false,
            default: () => {
                // Tu función por defecto aquí
                // Puedes devolver una función vacía o cualquier otra lógica por defecto
            },
        },
        heightTable: {
            type: String,
            default: "78vh",
        },
        functionChipsStateGeneral: {
            type: Function,
            required: false,
            default: () => {
                // Tu función por defecto aquí
                // Puedes devolver una función vacía o cualquier otra lógica por defecto
            },
        },
        hideBottomTable: {
            type: Boolean,
            default: false,
        },
        textSizeColumn: {
            type: Number,
            default: 10,
        },
        activateSelector: {
            type: Boolean,
            default: false,
        },
        typeSelection: {
            type: String,
            default: "none",
        },
        selectedProps: {
            type: Array,
            default: () => [],
        },
        filterColumns: {
            type: Boolean,
            default: false,
        },
        filterCheckColumns: {
            type: Boolean,
            default: true,
        },
    },
    setup(props, { emit }) {
        const editDialog = ref(false);
        const editValue = ref("");
        const selectedRow = ref(null);

        const selectedField = ref("");
        const labelSelectedField = ref("Editar Valor");

        const openEditDialog = (row, field) => {
            selectedRow.value = row;
            selectedField.value = field; // Almacenar el nombre del campo
            editValue.value = row[field]; // Asignar el valor del campo seleccionado para editarlo
            labelSelectedField.value = `Valor de ${field}`;
            editDialog.value = props.dataUserLogin.id_role_login == TypeRol.ADMIN ? true : false;
        };

        const saveEdit = async () => {
            // Asumiendo que updateFieldValueFunction realiza la actualización en el backend
            // y espera como parámetros el id de la fila, el campo a actualizar y el nuevo valor.
            if (props.updateFieldValueFunction && selectedRow.value && selectedField.value) {
                try {
                    if (editValue.value.length >= 1) {
                        await props.updateFieldValueFunction(selectedRow.value, selectedField.value, editValue.value);
                        // Encuentra la fila en auxOriginalRows y actualiza el valor directamente
                        const rowToUpdate = auxOriginalRows.value.find(
                            (r) => r.id_solicitud_general === selectedRow.value.id_solicitud_general
                        );
                        if (rowToUpdate) {
                            rowToUpdate[selectedField.value] = editValue.value;
                        }
                        editDialog.value = false; // Cierra el diálogo
                    }
                } catch (error) {
                    console.error("Error al actualizar el valor:", error);
                    // Manejar el error (mostrar un mensaje al usuario, por ejemplo)
                }
            }
        };

        const filter = ref("");
        // const dataTable = inject('dataTable');

        const userConfirm = reactive({
            show_confirm_modal: false,
            message: "",
            result: false,
        });

        const showConfirmStateModal = (user) => {
            props.editStateFuction(user);
        };

        const showFormGeneralModal = (data) => {
            props.editfuction(data);
        };
        const redirectAction = (data) => {
            props.editProfileFuction(data);
        };
        // const selectRow = (event, row) => {

        // }
        const handleHeaderClick = (column, props) => {
            //logia para el click derecho
        };
        const getUniqueValues = (fieldName) => {
            const uniqueValuesSet = new Set();

            props.dataTable.rows.forEach((row) => {
                if (row[fieldName] !== undefined && row[fieldName] !== null) {
                    uniqueValuesSet.add(row[fieldName]);
                }
            });

            return Array.from(uniqueValuesSet);
        };
        //----------------- para el filtro del search-----------------
        const arraycolumnFilter = ref([]);
        const auxOriginalRows = ref([]);
        const auxFilter = ref([]);
        const applyFilters = (field) => {
            if (arraycolumnFilter.value.length === 0) {
                auxOriginalRows.value = props.dataTable.rows;
            } else {
                auxFilter.value = props.dataTable.rows;
                auxOriginalRows.value = auxFilter.value.filter((row) => arraycolumnFilter.value.includes(row[field]));
            }
        };

        //------------------ fin del filtro del serach-----------------
        //--------------------Ordenar------------------------------------
        // const rowsTable = ref();
        const fieldSelect = ref(null);
        const sortColumn = (valor, propsLabel) => {
            fieldSelect.value = propsLabel;
            if (valor) {
                auxOriginalRows.value.sort((a, z) => {
                    const valueA = typeof a[propsLabel] === "string" ? a[propsLabel] : String(a[propsLabel]);
                    const valueZ = typeof z[propsLabel] === "string" ? z[propsLabel] : String(z[propsLabel]);
                    if (typeof a[propsLabel] === "number" && typeof z[propsLabel] === "number") {
                        return valueA - valueZ;
                    } else {
                        return valueA.localeCompare(valueZ);
                    }
                });
            } else {
                auxOriginalRows.value.sort((a, z) => {
                    const valueA = typeof a[propsLabel] === "string" ? a[propsLabel] : String(a[propsLabel]);
                    const valueZ = typeof z[propsLabel] === "string" ? z[propsLabel] : String(z[propsLabel]);
                    if (typeof a[propsLabel] === "number" && typeof z[propsLabel] === "number") {
                        return valueZ - valueA;
                    } else {
                        return valueZ.localeCompare(valueA);
                    }
                });
            }
        };

        //--------------------fin Ordenar---------------------------------
        //-------------------------------------Accion de doble click--------
        // const handleRowDblClick = (evt, row) => {
        //     // Lógica de la acción que quieres realizar al hacer doble clic en la fila
        //     console.log("Doble clic en la fila:", row);
        //     // Aquí puedes llamar a la función o realizar cualquier acción que desees
        //     // por ejemplo: props.editfuction(row);
        // };

        //------------------------------------------------------------------
        const textSize = ref(props.textSizeColumn);
        const truncateContent = (content) => {
            return content.slice(0, textSize.value) + "...";
        };

        const pagination = ref();
        watch(arraycolumnFilter, () => {
            auxFilter.value = auxOriginalRows.value;
        });
        watch(props.dataTable, () => {
            auxOriginalRows.value = props.dataTable.rows;
            pagination.value = props.dataTable.pagination;
        });
        onMounted(() => {
            //sdsd
        });
        //---------------------------------Inicio selector de filas en la tabla---------------------------

        const selected = ref([]);

        watch(
            () => props.selectedProps,
            (newValue, oldValue) => {
                selected.value = newValue;
            }
        );
        const getSelectedString = () => {
            return selected.value.length === 0
                ? ""
                : `${selected.value.length} Seleccionado${selected.value.length > 1 ? "s" : ""} de ${
                      auxOriginalRows.value.length
                  }`;
        };
        const $q = useQuasar();
        const updateSelection = (value) => {
            // emit("updateSelection", value);
            if (value.length > 50) {
                // Opción 1: Mostrar un mensaje de advertencia y no actualizar la selección
                // Puedes usar QNotify de Quasar para mostrar un mensaje de advertencia, o cualquier otro método que prefieras
                $q.notify({
                    color: "negative",
                    position: "bottom",
                    message: "Solo puedes seleccionar hasta 50 elementos.",
                    icon: "warning",
                    timeout: 500,
                });

                // Opción 2: Revertir a la última selección válida (los primeros 4 elementos seleccionados)
                // Esto asignará solo los primeros 4 elementos seleccionados y los demás serán ignorados
                value = selected.value.slice(0, 50); // Asume que `selected` es tu variable reactiva para almacenar la selección
            }

            // Emitir el evento con el valor actualizado (ya sea limitado a 4 o no, dependiendo de tu elección anterior)
            emit("updateSelection", value);

            // Asegurarte de actualizar el estado local para reflejar la selección actual (si decides limitar la selección a 4)
            selected.value = value;
        };
        //------------------------------------------Fin del selector---------------------------------------
        //------------------------------------Pintar y seleccionar fila----------------------------------
        const toggleRowSelection = (row) => {
            if (row.selected) {
                row.selected = false; // Si ya estaba seleccionada, la deseleccionamos
            } else {
                auxOriginalRows.value.forEach((row) => {
                    row.selected = false; // Deseleccionamos todas las filas
                });
                row.selected = true; // Luego seleccionamos la fila deseada
            }
        };
        //----------------------------Fin pintar y seleccionar-------------------------------------------
        // color de la fleha seleccionada al ordenar
        const getColumnColor = (field) => {
            return fieldSelect.value === field ? "orange-5" : "primary";
        };
        //--------------------------------inicio para buscador de cada columna---------------------------
        const inputSearchColumn = ref(null);
        const searchColumnFilter = (data, field) => {
            inputSearchColumn.value = data;

            if (inputSearchColumn.value == null || inputSearchColumn.value.trim() == "") {
                auxOriginalRows.value = props.dataTable.rows;
            } else {
                auxFilter.value = props.dataTable.rows;
                if (data && data.trim()) {
                    auxOriginalRows.value = auxFilter.value.filter((obj) => {
                        // Convierte el valor del campo a minúsculas para una comparación insensible a mayúsculas y minúsculas
                        const value = obj[field].toString().toLowerCase();
                        // Convierte el término de búsqueda a minúsculas
                        const term = data.toLowerCase().trim();
                        // Retorna true si el valor del campo incluye el término de búsqueda
                        return value.includes(term);
                    });
                }
            }
        };

        //--------------------------------Fin de buscador de cada columna---------------------------------
        return {
            ...toRefs(userConfirm),
            // dataTable,
            filter,
            showConfirmStateModal,
            showFormGeneralModal,
            redirectAction,
            handleHeaderClick,
            right2: ref(false),
            arraycolumnFilter,
            getUniqueValues,
            sortColumn,
            applyFilters,
            auxOriginalRows,
            pagination,
            truncateContent,
            textSize,
            selected,
            getSelectedString,
            updateSelection,
            toggleRowSelection,
            // formulario de edicion de datos
            editDialog,
            editValue,
            openEditDialog,
            saveEdit,
            labelSelectedField,
            validateByTwoCaracterNames,
            getColumnColor,
            searchColumnFilter,
            inputSearchColumn,
        };
    },
    methods: {
        emitEvent(props) {
            this.$emit("request", props);
        },
    },
};
</script>

<style>
.horizontal-checkbox-container {
    display: flex;

    flex-direction: column;
}

.loadingPaginate {
    padding-left: 50%;
}
.selected-row {
    background-color: rgb(156, 156, 215) !important;
    color: white !important;
}
</style>

<style lang="sass">
.my-sticky-header-table-general
  /* height or max-height is important */
  height: 310px

  .q-table__top,
  .q-table__bottom,
  thead tr:first-child th
    background-color: white
    color: black

  thead tr th
    position: sticky
    z-index: 1
  thead tr:first-child th
    top: 0

  /* this is when the loading indicator appears */
  &.q-table--loading thead tr:last-child th
    /* height of all previous header rows */
    top: 100%

  /* prevent scrolling behind sticky top row on focus */
  tbody
    /* height of all previous header rows */
    scroll-margin-top: 48px
</style>
