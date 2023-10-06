# Datagrid Inline Editing

Allowing users to conveniently and swiftly edit data in bulk can be a challenge for designers and developers alike. 

https://github.com/stadium-software/datagrid-inline-edit/assets/2085324/3119cdca-7d0f-44a8-8193-d792a2712a27

## Application Setup
1. Check the *Enable Style Sheet* checkbox in the application properties

## Database, Connector and DataGrid
1. Use the instructions from [this repo](https://github.com/stadium-software/samples-database) to setup the database and DataGrid for this sample

## Global Script Setup
1. Create a Global Script called "InlineEditing"
2. Add four input parameters to the Global Script
   1. DataGridClass
   2. FormFields
   3. IdentityColumnHeader
   4. ButtonClassName
3. Drag a *JavaScript* action into the script
4. Add the Javascript below into the JavaScript code property (ignore the validation error message "Invalid script was detected")
```
let scope = this;
let editing = false;
let dgClass = "." + ~.Parameters.Input.DataGridClass;
let dgParent = document.querySelector(dgClass);
if (!dgParent) dgParent = document.querySelector(".data-grid-container");
let dg = dgParent.querySelector("table");
dg.classList.add("stadium-inline-edit-datagrid");
let rowFormFields = ~.Parameters.Input.FormFields;
let IDColumn = ~.Parameters.Input.IdentityColumnHeader;
let editButtonParent = document.querySelector("." + ~.Parameters.Input.ButtonClassName);
let editButton = document.querySelector("." + ~.Parameters.Input.ButtonClassName + " button");
let buttonBar = document.createElement("div");
let rows = dg.querySelectorAll("tbody tr");
let IDColNo;
let options = {
    characterData: true,
    attributes: false,
    childList: true,
    subtree: true,
    characterDataOldValue: true,
},
observer = new MutationObserver(resetDataGrid);

function initForm(){
    let formFields = enrichFormFields(rowFormFields);
    let result = formFields.find((obj) => {
        return obj.type == "Identity";
    });
    IDColNo = result.colNo;

    let form = document.querySelector(".datagrid-inline-edit-form");
    if (!form) {
        let container = document.querySelector('div.container');
        form = document.createElement('form');
        form.classList.add("datagrid-inline-edit-form");
        container.parentNode.insertBefore(form, container);
        form.appendChild(container);
        form.addEventListener("submit", saveButtonClick);
    }

    for (let j = 0; j < rows.length; j++){
        for (let i = 0; i < formFields.length; i++) {
            rows[j].classList.remove("edited");
            let name = formFields[i].name;
            let colNo = formFields[i].colNo;
            let value = rows[j].querySelector("td:nth-child(" + formFields[i].colNo + ")").innerText;
            let type = formFields[i].type;
            let data = formFields[i].data;
            let min = formFields[i].min;
            let max = formFields[i].max;
            let required = formFields[i].required;
            let el;
            if (type == "Identity") {
                el = document.createElement("input");
                el.value = value;
                el.classList.add("visually-hidden");
                el.setAttribute("stadium-form-name", name);
            } else if (type == "date") {
                el = document.createElement("input");
                el.setAttribute("type", "date");
                el.classList.add("form-control");
                if (min) {
                    let dmin = new Date(min);
                    min = dmin.getFullYear() + '-' + ('0' + (dmin.getMonth() + 1)).slice(-2) + '-' + ('0' + dmin.getDate()).slice(-2);
                    el.setAttribute("min", min);
                }
                if (max) {
                    let dmax = new Date(max);
                    max = dmax.getFullYear() + '-' + ('0' + (dmax.getMonth() + 1)).slice(-2) + '-' + ('0' + dmax.getDate()).slice(-2);
                    el.setAttribute("max", max);
                }
                el.setAttribute("stadium-form-name", name);
                let d = new Date(value);
                el.value = d.getFullYear() + '-' + ('0' + (d.getMonth() + 1)).slice(-2) + '-' + ('0' + d.getDate()).slice(-2);
            } else if (type == "number") {
                el = document.createElement("input");
                el.setAttribute("type", "number");
                if (min) el.setAttribute("min", min);
                if (max) el.setAttribute("max", max);
                el.setAttribute("onkeydown", "return event.keyCode !== 69");
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.classList.add("form-control");
            } else if (type == "checkbox") {
                el = document.createElement("input");
                el.setAttribute("stadium-form-name", name);
                el.setAttribute("type", "checkbox");
                if (value == "true" || value == "Yes" || value == "1") {
                    el.setAttribute("checked", "");
                }
            } else if (type == "dropdown") {
                el = document.createElement("select");
                for (let j = 0; j < data.length; j++) {
                    let option = document.createElement("option");
                    option.text = data[j];
                    option.value = data[j];
                    el.appendChild(option);
                }
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.classList.add("form-control");
            } else { 
                el = document.createElement("input");
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.setAttribute("type", type);
                el.classList.add("form-control");
            }
            if (el) {
                el.classList.add("stadium-inline-form-control");
                let cell = rows[j].querySelector("td:nth-child(" + colNo + ")");
                if (type != "Identity") cell.querySelector("div").classList.add("visually-hidden");
                if (required) el.setAttribute("required", "");
                cell.appendChild(el);
            }
        }
    }
    prepareDataGrid();
    editing = true;
}
document.onkeydown = function (evt) {
    evt = evt || window.event;
    var isEscape = false;
    if ("key" in evt) {
        isEscape = (evt.key === "Escape" || evt.key === "Esc");
    } else {
        isEscape = (evt.keyCode === 27);
    }
    if (isEscape) {
        resetDataGrid();
    }
};

initForm();
observer.observe(dg, options);

/*--------------------------------------------------------------------------------------*/

function prepareDataGrid() { 
    buttonBar.classList.add("edit-button-bar");
    editButton.classList.add("visually-hidden");
    
    let saveButton = document.createElement("button");
    let cancelLink = document.createElement("a");
    saveButton.setAttribute("class", "btn btn-lg btn-default");
    saveButton.setAttribute("type", "submit");
    saveButton.innerText = "Save";

    cancelLink.setAttribute("class", "btn btn-lg btn-link");
    cancelLink.innerText = "Cancel";
    cancelLink.addEventListener("click", resetDataGrid);
    
    buttonBar.appendChild(saveButton);
    buttonBar.appendChild(cancelLink);
    editButtonParent.appendChild(buttonBar);

    let pagination = document.querySelector(".pagination");
    if (pagination) pagination.classList.add("visually-hidden");
    let datagridheader = document.querySelector(".data-grid-header");
    if (datagridheader) datagridheader.classList.add("visually-hidden");

    let arrHeadings = dg.querySelectorAll("thead th a");
    for (let i = 0; i < arrHeadings.length; i++) {
        arrHeadings[i].classList.add("visually-hidden");
        let heading = document.createElement("span");
        heading.innerText = arrHeadings[i].innerText;
        heading.classList.add("inline-edit-heading");
        arrHeadings[i].parentElement.appendChild(heading);
     }
}
function saveButtonClick(e) { 
    e.preventDefault();
    let arrGridData = [];
    for (let j = 0; j < rows.length; j++) {
        let formFields = rows[j].querySelectorAll("[stadium-form-name]");
        let arrData = [];
        for (let i = 0; i < formFields.length; i++) {
            let fieldValue = formFields[i].value;
            if (formFields[i].getAttribute("type") == "checkbox") fieldValue = formFields[i].checked;
            let field = { Name: formFields[i].getAttribute("stadium-form-name"), Value: fieldValue };
            arrData.push(field);
        }
        arrGridData.push(arrData);
    }
    resetDataGrid();
    scope.SaveGrid(arrGridData);
}
function enrichFormFields(data) {
    let arrHeadings = dg.querySelectorAll("thead th");
    for (let i = 0; i < arrHeadings.length; i++) {
        let heading = arrHeadings[i].querySelector("a");
        if (heading) {
            let needle = heading.textContent.toLowerCase();
            let index = data.findIndex((col) => col.name.toLowerCase() == needle);
            if (index > -1) {
                data[index].colNo = i + 1;
            } else if (IDColumn.toLowerCase() == arrHeadings[i].innerText.toLowerCase()) {
                data.push({ name: IDColumn, colNo: i + 1, type: "Identity" });
            }
        }
    }
    return data;
}
function resetDataGrid() { 
    observer.disconnect();
    let allFields = dg.querySelectorAll(".stadium-inline-form-control");
    for (let i = 0; i < allFields.length; i++) { 
        allFields[i].remove();
    }
    let visuallyHidden = dgParent.querySelectorAll(".visually-hidden");
    for (let i = 0; i < visuallyHidden.length; i++) { 
        visuallyHidden[i].classList.remove("visually-hidden");
    }
    editButton.classList.remove("visually-hidden");
    buttonBar.remove();
    let arrHeadings = dg.querySelectorAll(".inline-edit-heading");
    for (let i = 0; i < arrHeadings.length; i++) {
        arrHeadings[i].remove();
     }
}
```

## Page-Script Setup
1. Create a Script inside of the Page called "SaveGrid"
2. Add one input parameter to the Script
   1. UpdatedRows
3. Drag a *Notification* action into the script
4. In the *Message* property, select the *UpdatedRows* parameter from the *Script Input Parameters* category

## Page Setup
1. Drag a *Button* control into the page and enter some text (e.g. Enable Inline Editing)
2. Add a class of your choosing to the *Button* *Classes* property (e.g. datagrid-inline-edit-button)
3. Drag a *DataGrid* control to the page ([see above](#database-connector-and-datagrid))
4. Add a class of your choosing to the *DataGrid* *Classes* property (e.g datagrid-inline-edit)

## Page.Load Event Setup
1. Populate the DataGrid with data ([see above](#database-connector-and-datagrid))

## Type Setup
1. Create a *Type* called "FormField"
2. Add the following properties to the type
   1. "name" (Any)
   2. "type" (Any)
   3. "required" (Any)
   4. "min" (Any)
   5. "max" (Any)
   6. "data" (List)
      1. "Item" (Any)

![Form Fields](images/FormFieldType.png)

## Button.Click Event Setup
1. Drag a *List* action into the event script and name the List "FormFields"
2. Set the List *Item Type* property to "Types.FormField"
3. Define the editable columns of your datagrid and their form fields
   1. name: The heading of the column. **NOTE: The column header might contain spaces that your database column does not contain!**
   2. type: The type of the column. Supported are
      1. text
      2. date
      3. number
      4. checkbox
      5. dropdown
      6. email
   3. required: A boolean (add "true" if required)
   4. min: A minimum value for number or date columns
   5. max: A maximum value for number or date columns
   6. data: A simple list of values for dropdowns (see example below)
```
= [{
	"name": "End Date",
	"type": "date",
	"required": "true"
},{
	"name": "First Name",
	"type": "text"
},{
	"name": "Last Name",
	"type": "text"
},{
	"name": "No of Children",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "No of Pets",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "Start Date",
	"type": "date",
	"min": "01-01-2010",
	"max": "01-01-2024"
},{
	"name": "Healthy",
	"type": "checkbox"
},{
	"name": "Happy",
	"type": "checkbox"
},{
	"name": "Subscription",
	"type": "dropdown",
	"data": ["", "Subscribed", "Unsubscribed", "No data"],
	"required": "true"
}]
```
4. Drag the Global Script called "InlineEditing" into the event script
5. Complete the Input properties for the script
   1. DataGridClass: The classname you assigned to the *DataGrid*
   2. FormFields: The *List* called "FormFields"
   3. IdentityColumnHeader: The header of the column that uniquely identifies each row (e.g. ID). **NOTE: The column header might contain spaces that your database column does not contain!**
   4. ButtonClassName: The classname you assigned to the *Button*

![Inline Editing Input Parameters](images/InlineEditingInputParameters.png)

## Customising CSS
1. Open the CSS file called [*datagrid-inline-edit-variables.css*](datagrid-inline-edit-variables.css) from this repo
2. Adjust the variables in the *:root* element as you see fit

## Applying the CSS

**Stadium 6.6 or higher**
1. Create a folder called *CSS* inside of your Embedded Files in your application
2. Drag the two CSS files from this repo [*datagrid-inline-edit-variables.css*](datagrid-inline-edit-variables.css) and [*datagrid-inline-edit.css*](datagrid-inline-edit.css) into that folder
3. Paste the link tags below into the *head* property of your application
```
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-inline-edit.css">
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-inline-edit-variables.css">
``` 

**Versions lower than 6.6**
1. Copy the CSS from the two css files into the Stylesheet in your application

## CSS Upgrading
To upgrade the CSS in this module, follow the [steps outlined in this repo](https://github.com/stadium-software/samples-upgrading)
