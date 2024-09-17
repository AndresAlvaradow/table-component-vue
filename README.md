# table-component-vue

#data structure
```
const table = reactive({
    rows: [] as Person[],
    users: [] as Person[],
    columns: [
        { name: 'index', label: '#', field: 'index', align: "left", },
        {
            name: "nombres",
            label: "Nombres",
            field: "nombres",
            align: "left",
            sortable: true,
            columnPopUpFilter: true,
        },
        {
            name: "apellidos",
            label: "Apellido",
            field: "apellidos",
            align: "left",
            sortable: true,
        },
        {
            name: "cedula",
            label: "Cédula",
            field: "cedula",
            align: "right",
            classes: "number-font",
            sortable: true,
            columnPopUpFilter: true,
        },
        {
            name: "celular",
            label: "Celular",
            field: "celular",
            align: "right",
            classes: "number-font",
            sortable: true,
            columnPopUpFilter: true,
        },
        {
            name: "email",
            label: "Correo",
            field: "correo",
            align: "left",
            sortable: true,
            columnPopUpFilter: true,

        },
        {
            name: "actionsOpcion",
            label: "Acciones",
            field: "",
            align: "center",
        },
    ] as QTableProps["columns"],
    pagination: {
        sortBy: "",
        descending: false,
        page: 1,
        rowsPerPage: 10,
        rowsNumber: 10,
    },
    loading: false,
});
```
# Llamado
```
  <TableComponenGeneral
title="Usuarios"
@request="onRequest"
:loading="loading"
:perPageOptions="perPageOptions"
:editfuction="(data) => showFormUserModal(data)"
:editStateFuction="(data) => showConfirmModal(data)"
rowsPerLabel="Filas por página:"
:loadPagination="loading"
:dataTable="table"
toolTipsSearch="Nombre, Cédula"
:handleRowDblClick="(evt, data) => openModalDetail(evt, data)"
:dataOption="dataAccionOpcion"
:filterCheckColumns="false"
:filterColumns="true"
/>
```
