// Global variables
let currentTab = 'setup';
let projectsData = [];
let lineItemsData = [];
let currentProject = null;
let currentEstimateItems = [];
let availableLineItems = [];

// Initialize app
document.addEventListener('DOMContentLoaded', function() {
    showTab('setup');
});

// Tab management
function showTab(tabName) {
    // Update nav
    document.querySelectorAll('.nav-tab').forEach(tab => tab.classList.remove('active'));
    document.querySelector(`[onclick="showTab('${tabName}')"]`).classList.add('active');
    
    // Update content
    document.querySelectorAll('.tab-content').forEach(content => content.classList.remove('active'));
    document.getElementById(`${tabName}-tab`).classList.add('active');
    
    currentTab = tabName;
    
    // Load data for tab
    if (tabName === 'projects') {
        loadProjects();
    } else if (tabName === 'line-items') {
        loadLineItems();
        loadCategories();
    } else if (tabName === 'estimates') {
        loadProjectsForEstimate();
    }
}

// Message handling
function showMessage(text, type = 'info', container = 'messages') {
    const messagesDiv = document.getElementById(container);
    if (!messagesDiv) return;
    
    const messageDiv = document.createElement('div');
    messageDiv.className = `message ${type}`;
    messageDiv.innerHTML = `<strong>${new Date().toLocaleTimeString()}:</strong> ${text}`;
    messagesDiv.appendChild(messageDiv);
    messagesDiv.scrollTop = messagesDiv.scrollHeight;
    
    // Auto-remove after 10 seconds
    setTimeout(() => {
        if (messageDiv.parentNode) {
            messageDiv.parentNode.removeChild(messageDiv);
        }
    }, 10000);
}

// Loading state
function setLoading(element, isLoading) {
    if (isLoading) {
        element.classList.add('loading');
    } else {
        element.classList.remove('loading');
    }
}

// Modal management
function showModal(modalId) {
    document.getElementById(modalId).classList.add('active');
}

function hideModal(modalId) {
    document.getElementById(modalId).classList.remove('active');
    // Clear form
    const form = document.querySelector(`#${modalId} form`);
    if (form) form.reset();
}

// Setup commands
async function runCommand(command) {
    try {
        showMessage(`Running command: ${command}`, 'info');
        const response = await fetch(`/admin/run/${command}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' }
        });
        const result = await response.json();
        showMessage(result.message, result.status);
        
        if (result.status === 'success' && (command.includes('seed') || command === 'init-db' || command === 'clear-all')) {
            // Refresh data if we seeded or cleared something
            if (currentTab === 'projects') loadProjects();
            if (currentTab === 'line-items') loadLineItems();
        }
    } catch (error) {
        showMessage(`Error: ${error.message}`, 'error');
    }
}

async function checkAPI() {
    try {
        const response = await fetch('/api/projects');
        const projects = await response.json();
        showMessage(`API is connected! Found ${projects.length} projects.`, 'success');
    } catch (error) {
        showMessage(`API Error: ${error.message}. Is the server running?`, 'error');
    }
}

async function loadSystemStats() {
    try {
        const [projectsRes, itemsRes] = await Promise.all([
            fetch('/api/projects'),
            fetch('/api/line-items')
        ]);
        
        const projects = await projectsRes.json();
        const items = await itemsRes.json();
        
        showMessage(`System Stats: ${projects.length} projects, ${items.length} line items`, 'success');
    } catch (error) {
        showMessage(`Error loading stats: ${error.message}`, 'error');
    }
}

// ------------------- Projects Management -------------------
async function loadProjects() {
    const container = document.getElementById('projects-container');
    setLoading(container, true);
    
    try {
        const response = await fetch('/api/projects');
        if (!response.ok) throw new Error('Network response was not ok');
        projectsData = await response.json();
        
        if (projectsData.length === 0) {
            container.innerHTML = `
                <div class="empty-state">
                    <h3>No Projects Found</h3>
                    <p>Click "Add New Project" or "Seed Projects" to get started.</p>
                </div>
            `;
        } else {
            displayProjects(projectsData);
        }
    } catch (error) {
        container.innerHTML = `<div class="message error">Error loading projects: ${error.message}</div>`;
    } finally {
        setLoading(container, false);
    }
}

function displayProjects(projects) {
    const container = document.getElementById('projects-container');
    const html = `
        <table class="table">
            <thead>
                <tr>
                    <th>Project Name</th>
                    <th>Claim #</th>
                    <th>Customer</th>
                    <th>Status</th>
                    <th>Items</th>
                    <th>Last Updated</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                ${projects.map(project => `
                    <tr>
                        <td><strong>${project.ProjectName}</strong></td>
                        <td>${project.ClaimNumber || '-'}</td>
                        <td>${project.CustomerName || '-'}</td>
                        <td><span class="status-badge status-${project.ProjectStatus.toLowerCase().replace(' ', '-')}">${project.ProjectStatus}</span></td>
                        <td>${project.LineItemCount || 0}</td>
                        <td>${project.LastUpdated ? new Date(project.LastUpdated).toLocaleDateString() : '-'}</td>
                        <td>
                            <button class="btn btn-sm" onclick="editProject(${project.ProjectID})">Edit</button>
                            <button class="btn btn-sm btn-warning" onclick="viewEstimate(${project.ProjectID})">Estimate</button>
                            <button class="btn btn-sm btn-danger" onclick="deleteProject(${project.ProjectID})">Delete</button>
                        </td>
                    </tr>
                `).join('')}
            </tbody>
        </table>
    `;
    container.innerHTML = html;
}

function searchProjects() {
    const searchTerm = document.getElementById('project-search').value.toLowerCase();
    const filteredProjects = projectsData.filter(project => 
        (project.ProjectName && project.ProjectName.toLowerCase().includes(searchTerm)) ||
        (project.CustomerName && project.CustomerName.toLowerCase().includes(searchTerm)) ||
        (project.ClaimNumber && project.ClaimNumber.toLowerCase().includes(searchTerm))
    );
    displayProjects(filteredProjects);
}

function showProjectModal(project = {}) {
    document.getElementById('project-modal-title').textContent = project.ProjectID ? 'Edit Project' : 'Add New Project';
    document.getElementById('project-id').value = project.ProjectID || '';
    document.getElementById('project-name').value = project.ProjectName || '';
    document.getElementById('claim-number').value = project.ClaimNumber || '';
    document.getElementById('project-status').value = project.ProjectStatus || 'Draft';
    document.getElementById('customer-name').value = project.CustomerName || '';
    document.getElementById('property-address').value = project.PropertyAddress || '';
    document.getElementById('contact-phone').value = project.ContactPhone || '';
    document.getElementById('contact-email').value = project.ContactEmail || '';
    document.getElementById('insurance-company').value = project.InsuranceCompany || '';
    document.getElementById('policy-number').value = project.PolicyNumber || '';
    document.getElementById('project-notes').value = project.ProjectNotes || '';
    showModal('project-modal');
}

async function saveProject() {
    const id = document.getElementById('project-id').value;
    const project = {
        ProjectName: document.getElementById('project-name').value,
        ClaimNumber: document.getElementById('claim-number').value,
        ProjectStatus: document.getElementById('project-status').value,
        CustomerName: document.getElementById('customer-name').value,
        PropertyAddress: document.getElementById('property-address').value,
        ContactPhone: document.getElementById('contact-phone').value,
        ContactEmail: document.getElementById('contact-email').value,
        InsuranceCompany: document.getElementById('insurance-company').value,
        PolicyNumber: document.getElementById('policy-number').value,
        ProjectNotes: document.getElementById('project-notes').value
    };
    
    if (!project.ProjectName) {
        showMessage('Project Name is required!', 'error', 'project-modal .modal-body');
        return;
    }

    try {
        const method = id ? 'PUT' : 'POST';
        const url = id ? `/api/projects/${id}` : '/api/projects';
        const response = await fetch(url, {
            method: method,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(project)
        });
        
        const result = await response.json();
        if (result.status === 'success') {
            hideModal('project-modal');
            showMessage(result.message, 'success');
            loadProjects();
        } else {
            showMessage(result.message, 'error', 'project-modal .modal-body');
        }
    } catch (error) {
        showMessage(`Error saving project: ${error.message}`, 'error', 'project-modal .modal-body');
    }
}

async function editProject(id) {
    try {
        const response = await fetch(`/api/projects/${id}`);
        const project = await response.json();
        showProjectModal(project);
    } catch (error) {
        showMessage(`Error fetching project: ${error.message}`, 'error');
    }
}

async function deleteProject(id) {
    if (!confirm('Are you sure you want to delete this project and all its associated estimate items?')) return;
    try {
        const response = await fetch(`/api/projects/${id}`, { method: 'DELETE' });
        const result = await response.json();
        if (result.status === 'success') {
            showMessage(result.message, 'success');
            loadProjects();
        } else {
            showMessage(result.message, 'error');
        }
    } catch (error) {
        showMessage(`Error deleting project: ${error.message}`, 'error');
    }
}

// ------------------- Line Items Management -------------------
async function loadLineItems() {
    const container = document.getElementById('line-items-container');
    setLoading(container, true);
    
    try {
        const response = await fetch('/api/line-items');
        if (!response.ok) throw new Error('Network response was not ok');
        lineItemsData = await response.json();
        availableLineItems = [...lineItemsData]; // Copy for estimates dropdown
        
        if (lineItemsData.length === 0) {
            container.innerHTML = `
                <div class="empty-state">
                    <h3>No Line Items Found</h3>
                    <p>Click "Add New Line Item" or "Seed Line Items" to get started.</p>
                </div>
            `;
        } else {
            displayLineItems(lineItemsData);
        }
    } catch (error) {
        container.innerHTML = `<div class="message error">Error loading line items: ${error.message}</div>`;
    } finally {
        setLoading(container, false);
    }
}

function displayLineItems(items) {
    const container = document.getElementById('line-items-container');
    const html = `
        <table class="table">
            <thead>
                <tr>
                    <th>Description</th>
                    <th>Unit</th>
                    <th>Cost</th>
                    <th>Category</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                ${items.map(item => `
                    <tr>
                        <td>${item.Description}</td>
                        <td>${item.UnitOfMeasure}</td>
                        <td>$${(item.UnitCost || 0).toFixed(2)}</td>
                        <td>${item.Category || '-'}</td>
                        <td>
                            <button class="btn btn-sm" onclick="editLineItem(${item.LineItemID})">Edit</button>
                            <button class="btn btn-sm btn-danger" onclick="deleteLineItem(${item.LineItemID})">Delete</button>
                        </td>
                    </tr>
                `).join('')}
            </tbody>
        </table>
    `;
    container.innerHTML = html;
}

function searchLineItems() {
    const searchTerm = document.getElementById('lineitem-search').value.toLowerCase();
    const categoryFilter = document.getElementById('category-filter').value;
    const filteredItems = lineItemsData.filter(item => {
        const matchesSearch = item.Description.toLowerCase().includes(searchTerm);
        const matchesCategory = categoryFilter === '' || (item.Category && item.Category.toLowerCase() === categoryFilter.toLowerCase());
        return matchesSearch && matchesCategory;
    });
    displayLineItems(filteredItems);
}

function filterByCategory() {
    searchLineItems(); // The search function handles both search and filter
}

async function loadCategories() {
    try {
        const response = await fetch('/api/line-items/categories');
        const categories = await response.json();
        const select = document.getElementById('category-filter');
        select.innerHTML = '<option value="">All Categories</option>';
        categories.forEach(cat => {
            const option = document.createElement('option');
            option.value = cat;
            option.textContent = cat;
            select.appendChild(option);
        });
    } catch (error) {
        console.error('Failed to load categories:', error);
    }
}

function showLineItemModal(item = {}) {
    document.getElementById('lineitem-modal-title').textContent = item.LineItemID ? 'Edit Line Item' : 'Add New Line Item';
    document.getElementById('lineitem-id').value = item.LineItemID || '';
    document.getElementById('lineitem-description').value = item.Description || '';
    document.getElementById('lineitem-unit').value = item.UnitOfMeasure || 'Each';
    document.getElementById('lineitem-cost').value = item.UnitCost || 0;
    document.getElementById('lineitem-category').value = item.Category || '';
    document.getElementById('lineitem-notes').value = item.Notes || '';
    showModal('lineitem-modal');
}

async function saveLineItem() {
    const id = document.getElementById('lineitem-id').value;
    const item = {
        Description: document.getElementById('lineitem-description').value,
        UnitOfMeasure: document.getElementById('lineitem-unit').value,
        UnitCost: parseFloat(document.getElementById('lineitem-cost').value),
        Category: document.getElementById('lineitem-category').value,
        Notes: document.getElementById('lineitem-notes').value
    };
    
    if (!item.Description || isNaN(item.UnitCost)) {
        showMessage('Description and Unit Cost are required!', 'error', 'lineitem-modal .modal-body');
        return;
    }

    try {
        const method = id ? 'PUT' : 'POST';
        const url = id ? `/api/line-items/${id}` : '/api/line-items';
        const response = await fetch(url, {
            method: method,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(item)
        });
        
        const result = await response.json();
        if (result.status === 'success') {
            hideModal('lineitem-modal');
            showMessage(result.message, 'success');
            loadLineItems();
        } else {
            showMessage(result.message, 'error', 'lineitem-modal .modal-body');
        }
    } catch (error) {
        showMessage(`Error saving line item: ${error.message}`, 'error', 'lineitem-modal .modal-body');
    }
}

async function editLineItem(id) {
    try {
        const response = await fetch(`/api/line-items/${id}`);
        const item = await response.json();
        showLineItemModal(item);
    } catch (error) {
        showMessage(`Error fetching line item: ${error.message}`, 'error');
    }
}

async function deleteLineItem(id) {
    if (!confirm('Are you sure you want to delete this line item?')) return;
    try {
        const response = await fetch(`/api/line-items/${id}`, { method: 'DELETE' });
        const result = await response.json();
        if (result.status === 'success') {
            showMessage(result.message, 'success');
            loadLineItems();
        } else {
            showMessage(result.message, 'error');
        }
    } catch (error) {
        showMessage(`Error deleting line item: ${error.message}`, 'error');
    }
}

// ------------------- Estimates Management -------------------
async function loadProjectsForEstimate() {
    const select = document.getElementById('estimate-project-select');
    select.innerHTML = '<option value="">Select a project...</option>';
    
    try {
        const response = await fetch('/api/projects');
        const projects = await response.json();
        projects.forEach(project => {
            const option = document.createElement('option');
            option.value = project.ProjectID;
            option.textContent = project.ProjectName;
            select.appendChild(option);
        });
    } catch (error) {
        console.error('Failed to load projects for estimate:', error);
    }
}

async function loadProjectEstimate() {
    const projectId = document.getElementById('estimate-project-select').value;
    const container = document.getElementById('estimate-container');
    
    if (!projectId) {
        container.innerHTML = `
            <div class="empty-state">
                <h3>Select a Project</h3>
                <p>Choose a project from the dropdown above to view and manage its estimate</p>
            </div>
        `;
        currentProject = null;
        return;
    }

    setLoading(container, true);
    
    try {
        const projectRes = await fetch(`/api/projects/${projectId}`);
        const lineItemsRes = await fetch(`/api/projects/${projectId}/estimate`);
        
        currentProject = await projectRes.json();
        currentEstimateItems = await lineItemsRes.json();
        
        displayEstimate(currentProject, currentEstimateItems);
        loadAvailableLineItemsForEstimate();
    } catch (error) {
        container.innerHTML = `<div class="message error">Error loading estimate: ${error.message}</div>`;
    } finally {
        setLoading(container, false);
    }
}

function displayEstimate(project, items) {
    const container = document.getElementById('estimate-container');
    const totalCost = items.reduce((sum, item) => sum + (item.Quantity * item.UnitCost), 0);
    
    const html = `
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
            <h4>Estimate for: ${project.ProjectName}</h4>
            <h4>Total: $${totalCost.toFixed(2)}</h4>
        </div>
        <div style="display: flex; gap: 10px; margin-bottom: 20px;">
            <select id="line-item-select" class="form-control" style="flex: 2;">
                <option value="">Select a line item to add...</option>
            </select>
            <input type="number" id="line-item-quantity" class="form-control" placeholder="Quantity" value="1" min="1" style="flex: 1;">
            <button class="btn btn-success" onclick="addLineItemToEstimate()">Add to Estimate</button>
        </div>
        
        <table class="table">
            <thead>
                <tr>
                    <th>Description</th>
                    <th>Quantity</th>
                    <th>Unit</th>
                    <th>Unit Cost</th>
                    <th>Subtotal</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                ${items.length > 0 ? items.map(item => `
                    <tr>
                        <td>${item.Description}</td>
                        <td>${item.Quantity}</td>
                        <td>${item.UnitOfMeasure}</td>
                        <td>$${item.UnitCost.toFixed(2)}</td>
                        <td>$${(item.Quantity * item.UnitCost).toFixed(2)}</td>
                        <td>
                            <button class="btn btn-sm btn-danger" onclick="deleteEstimateItem(${item.EstimateItemID})">Remove</button>
                        </td>
                    </tr>
                `).join('') : `
                    <tr>
                        <td colspan="6" class="empty-state">No line items in this estimate. Add one above!</td>
                    </tr>
                `}
            </tbody>
        </table>
    `;
    container.innerHTML = html;
}

async function loadAvailableLineItemsForEstimate() {
    const select = document.getElementById('line-item-select');
    select.innerHTML = '<option value="">Select a line item to add...</option>';
    
    try {
        const response = await fetch('/api/line-items');
        availableLineItems = await response.json();
        availableLineItems.forEach(item => {
            const option = document.createElement('option');
            option.value = item.LineItemID;
            option.textContent = `${item.Description} ($${item.UnitCost.toFixed(2)}/${item.UnitOfMeasure})`;
            select.appendChild(option);
        });
    } catch (error) {
        console.error('Failed to load available line items:', error);
    }
}

async function addLineItemToEstimate() {
    const projectId = document.getElementById('estimate-project-select').value;
    const lineItemId = document.getElementById('line-item-select').value;
    const quantity = parseInt(document.getElementById('line-item-quantity').value);

    if (!projectId || !lineItemId || isNaN(quantity) || quantity <= 0) {
        showMessage('Please select a line item and enter a valid quantity.', 'error', 'estimate-container');
        return;
    }

    try {
        const itemToAdd = availableLineItems.find(item => item.LineItemID == lineItemId);
        if (!itemToAdd) throw new Error('Selected line item not found.');

        const newEstimateItem = {
            ProjectID: parseInt(projectId),
            LineItemID: parseInt(lineItemId),
            Quantity: quantity,
            UnitCost: itemToAdd.UnitCost
        };
        
        const response = await fetch('/api/estimates', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(newEstimateItem)
        });

        const result = await response.json();
        if (result.status === 'success') {
            showMessage(result.message, 'success', 'estimate-container');
            loadProjectEstimate(); // Reload the estimate to show the new item
        } else {
            showMessage(result.message, 'error', 'estimate-container');
        }
    } catch (error) {
        showMessage(`Error adding line item: ${error.message}`, 'error', 'estimate-container');
    }
}

async function deleteEstimateItem(estimateItemId) {
    if (!confirm('Are you sure you want to remove this item from the estimate?')) return;
    
    try {
        const response = await fetch(`/api/estimates/${estimateItemId}`, { method: 'DELETE' });
        const result = await response.json();
        if (result.status === 'success') {
            showMessage(result.message, 'success', 'estimate-container');
            loadProjectEstimate(); // Reload the estimate
        } else {
            showMessage(result.message, 'error', 'estimate-container');
        }
    } catch (error) {
        showMessage(`Error removing item: ${error.message}`, 'error', 'estimate-container');
    }
}

// Function to switch to the estimates tab and load a specific project
function viewEstimate(projectId) {
    showTab('estimates');
    setTimeout(() => {
        const select = document.getElementById('estimate-project-select');
        select.value = projectId;
        loadProjectEstimate();
    }, 100);
}