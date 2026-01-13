# Inventory Management System

## 📋 Project Overview
A comprehensive console-based inventory management system built with C# that allows users to efficiently track and manage product stock. This application provides essential features for product management, stock control, and inventory reporting with persistent data storage.

## ✨ Features
### Core Functionality
#### Product Management
 - Add new products with name, price, stock quantity, and category
 - Update existing product information
 - Remove products from inventory
 - View detailed product information
#### Stock Control
 - Increase stock quantity (restocking)
 - Decrease stock quantity (sales/fulfillment)
 - Set exact stock quantities
 - Track last update timestamps
#### Inventory Reporting
 - View all products with complete details
 - Generate low stock alerts with customizable thresholds
 - Calculate total inventory value
 - Search products by ID or name
#### Data Management
 - Automatic JSON file persistence
 - Data validation and error handling
 - Import/export capabilities

## 🛠️ Technology Stack
 - Language: C# (.NET Core/.NET 5+)
 - Data Persistence: JSON file storage
 - Serialization: System.Text.Json
 - Platform: Console Application

## 📁 Project Structure
```
InventoryManagementSystem/
├── Program.cs                 # Main application entry point
├── Product.cs                # Product entity class
├── InventoryManager.cs       # Core business logic
├── FileManager.cs           # Data persistence handler
├── ConsoleUI.cs             # User interface manager
├── README.md                # This documentation
└── inventory.json           # Data storage file (auto-generated)
```
```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.Linq;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace InventoryManagementSystem
{
    // Main Program Class
    class Program
    {
        static void Main(string[] args)
        {
            ConsoleUI ui = new ConsoleUI();
            ui.Run();
        }
    }

    // Product Class - Represents an inventory item
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
        public int StockQuantity { get; set; }
        public string Category { get; set; }
        public DateTime LastUpdated { get; set; }

        public Product()
        {
            LastUpdated = DateTime.Now;
        }

        public void DisplayInfo()
        {
            Console.WriteLine($"ID: {Id}");
            Console.WriteLine($"Name: {Name}");
            Console.WriteLine($"Price: ${Price:F2}");
            Console.WriteLine($"Stock: {StockQuantity}");
            Console.WriteLine($"Category: {Category}");
            Console.WriteLine($"Last Updated: {LastUpdated:yyyy-MM-dd HH:mm:ss}");
            Console.WriteLine(new string('-', 40));
        }

        public bool UpdateStock(int quantity)
        {
            if (StockQuantity + quantity < 0)
                return false;

            StockQuantity += quantity;
            LastUpdated = DateTime.Now;
            return true;
        }
    }

    // Inventory Manager - Core business logic
    public class InventoryManager
    {
        private List<Product> products;
        private int nextId;

        public InventoryManager()
        {
            products = new List<Product>();
            nextId = 1;
        }

        // Add a new product
        public Product AddProduct(string name, decimal price, int quantity, string category)
        {
            var product = new Product
            {
                Id = nextId++,
                Name = name,
                Price = price,
                StockQuantity = quantity,
                Category = category,
                LastUpdated = DateTime.Now
            };

            products.Add(product);
            return product;
        }

        // Remove a product by ID
        public bool RemoveProduct(int id)
        {
            var product = FindProductById(id);
            if (product != null)
            {
                products.Remove(product);
                return true;
            }
            return false;
        }

        // Find product by ID
        public Product FindProductById(int id)
        {
            return products.FirstOrDefault(p => p.Id == id);
        }

        // Find products by name (partial match)
        public List<Product> FindProductByName(string name)
        {
            return products.Where(p => p.Name.Contains(name, StringComparison.OrdinalIgnoreCase)).ToList();
        }

        // Update stock quantity
        public bool UpdateStock(int id, int quantity)
        {
            var product = FindProductById(id);
            if (product != null)
            {
                return product.UpdateStock(quantity);
            }
            return false;
        }

        // Get all products
        public List<Product> GetAllProducts()
        {
            return products.OrderBy(p => p.Id).ToList();
        }

        // Get products with low stock
        public List<Product> GetLowStockProducts(int threshold = 5)
        {
            return products.Where(p => p.StockQuantity <= threshold).OrderBy(p => p.StockQuantity).ToList();
        }

        // Calculate total inventory value
        public decimal CalculateTotalValue()
        {
            return products.Sum(p => p.Price * p.StockQuantity);
        }

        // Load products from list (for file loading)
        public void LoadProducts(List<Product> loadedProducts)
        {
            products = loadedProducts;
            nextId = products.Count > 0 ? products.Max(p => p.Id) + 1 : 1;
        }
    }

    // File Manager - Handles data persistence
    public class FileManager
    {
        private string filePath = "inventory.json";

        public bool SaveInventory(List<Product> products)
        {
            try
            {
                var options = new JsonSerializerOptions
                {
                    WriteIndented = true,
                    Converters = { new JsonStringEnumConverter() }
                };

                string json = JsonSerializer.Serialize(products, options);
                File.WriteAllText(filePath, json);
                return true;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error saving inventory: {ex.Message}");
                return false;
            }
        }

        public List<Product> LoadInventory()
        {
            try
            {
                if (File.Exists(filePath))
                {
                    string json = File.ReadAllText(filePath);
                    var products = JsonSerializer.Deserialize<List<Product>>(json);
                    return products ?? new List<Product>();
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error loading inventory: {ex.Message}");
            }
            return new List<Product>();
        }
    }

    // Console User Interface
    public class ConsoleUI
    {
        private InventoryManager inventory;
        private FileManager fileManager;

        public ConsoleUI()
        {
            inventory = new InventoryManager();
            fileManager = new FileManager();
        }

        public void Run()
        {
            LoadData();
            DisplayWelcomeMessage();

            bool exit = false;
            while (!exit)
            {
                DisplayMenu();
                string choice = GetUserInput("Enter your choice: ");

                switch (choice)
                {
                    case "1":
                        AddProduct();
                        break;
                    case "2":
                        ViewAllProducts();
                        break;
                    case "3":
                        UpdateStock();
                        break;
                    case "4":
                        RemoveProduct();
                        break;
                    case "5":
                        ViewLowStock();
                        break;
                    case "6":
                        CalculateTotalValue();
                        break;
                    case "7":
                        SearchProduct();
                        break;
                    case "8":
                        exit = true;
                        break;
                    default:
                        ShowMessage("Invalid choice. Please try again.", MessageType.Error);
                        break;
                }

                if (!exit)
                {
                    Console.WriteLine("\nPress any key to continue...");
                    Console.ReadKey();
                }
            }

            SaveData();
            ShowMessage("Inventory saved. Goodbye!", MessageType.Info);
        }

        private void LoadData()
        {
            var loadedProducts = fileManager.LoadInventory();
            if (loadedProducts.Any())
            {
                inventory.LoadProducts(loadedProducts);
                ShowMessage($"Loaded {loadedProducts.Count} products from file.", MessageType.Success);
            }
        }

        private void SaveData()
        {
            if (fileManager.SaveInventory(inventory.GetAllProducts()))
            {
                ShowMessage("Inventory saved successfully.", MessageType.Success);
            }
        }

        private void DisplayWelcomeMessage()
        {
            Console.Clear();
            Console.WriteLine(new string('=', 50));
            Console.WriteLine("       INVENTORY MANAGEMENT SYSTEM");
            Console.WriteLine(new string('=', 50));
            Console.WriteLine();
        }

        private void DisplayMenu()
        {
            Console.Clear();
            Console.WriteLine("MAIN MENU");
            Console.WriteLine(new string('-', 30));
            Console.WriteLine("1. Add New Product");
            Console.WriteLine("2. View All Products");
            Console.WriteLine("3. Update Stock");
            Console.WriteLine("4. Remove Product");
            Console.WriteLine("5. View Low Stock Alert");
            Console.WriteLine("6. Calculate Total Inventory Value");
            Console.WriteLine("7. Search Product");
            Console.WriteLine("8. Save and Exit");
            Console.WriteLine(new string('-', 30));
        }

        private void AddProduct()
        {
            Console.Clear();
            Console.WriteLine("ADD NEW PRODUCT");
            Console.WriteLine(new string('-', 30));

            string name = GetValidatedInput("Product Name: ", input => !string.IsNullOrWhiteSpace(input));
            decimal price = GetValidatedDecimal("Price: $");
            int quantity = GetValidatedInteger("Initial Stock Quantity: ");
            string category = GetValidatedInput("Category: ", input => !string.IsNullOrWhiteSpace(input));

            var product = inventory.AddProduct(name, price, quantity, category);
            ShowMessage($"Product added successfully! ID: {product.Id}", MessageType.Success);
        }

        private void ViewAllProducts()
        {
            Console.Clear();
            Console.WriteLine("ALL PRODUCTS IN INVENTORY");
            Console.WriteLine(new string('-', 50));

            var products = inventory.GetAllProducts();
            if (!products.Any())
            {
                ShowMessage("No products in inventory.", MessageType.Info);
                return;
            }

            foreach (var product in products)
            {
                product.DisplayInfo();
            }

            Console.WriteLine($"Total Products: {products.Count}");
        }

        private void UpdateStock()
        {
            Console.Clear();
            Console.WriteLine("UPDATE STOCK");
            Console.WriteLine(new string('-', 30));

            ViewAllProducts();
            
            int id = GetValidatedInteger("Enter Product ID to update: ");
            var product = inventory.FindProductById(id);

            if (product == null)
            {
                ShowMessage($"Product with ID {id} not found.", MessageType.Error);
                return;
            }

            Console.WriteLine($"Current stock for '{product.Name}': {product.StockQuantity}");
            Console.WriteLine("\nUpdate Options:");
            Console.WriteLine("1. Add Stock (Restock)");
            Console.WriteLine("2. Remove Stock (Sale)");
            Console.WriteLine("3. Set Exact Quantity");

            string choice = GetUserInput("Choose update type (1-3): ");
            int quantity;

            switch (choice)
            {
                case "1":
                    quantity = GetValidatedInteger("Enter quantity to add: ");
                    if (inventory.UpdateStock(id, quantity))
                    {
                        ShowMessage($"Stock updated successfully. New quantity: {product.StockQuantity + quantity}", MessageType.Success);
                    }
                    else
                    {
                        ShowMessage("Invalid quantity.", MessageType.Error);
                    }
                    break;
                case "2":
                    quantity = GetValidatedInteger("Enter quantity to remove: ");
                    if (inventory.UpdateStock(id, -quantity))
                    {
                        ShowMessage($"Stock updated successfully. New quantity: {product.StockQuantity - quantity}", MessageType.Success);
                    }
                    else
                    {
                        ShowMessage("Cannot remove more than available stock.", MessageType.Error);
                    }
                    break;
                case "3":
                    quantity = GetValidatedInteger("Enter new quantity: ");
                    int difference = quantity - product.StockQuantity;
                    if (inventory.UpdateStock(id, difference))
                    {
                        ShowMessage($"Stock updated successfully to {quantity}.", MessageType.Success);
                    }
                    else
                    {
                        ShowMessage("Invalid quantity.", MessageType.Error);
                    }
                    break;
                default:
                    ShowMessage("Invalid choice.", MessageType.Error);
                    break;
            }
        }

        private void RemoveProduct()
        {
            Console.Clear();
            Console.WriteLine("REMOVE PRODUCT");
            Console.WriteLine(new string('-', 30));

            ViewAllProducts();
            
            int id = GetValidatedInteger("Enter Product ID to remove: ");
            var product = inventory.FindProductById(id);

            if (product == null)
            {
                ShowMessage($"Product with ID {id} not found.", MessageType.Error);
                return;
            }

            Console.WriteLine($"Are you sure you want to remove '{product.Name}' (ID: {product.Id})?");
            Console.Write("Type 'YES' to confirm: ");
            string confirmation = Console.ReadLine();

            if (confirmation?.ToUpper() == "YES")
            {
                if (inventory.RemoveProduct(id))
                {
                    ShowMessage($"Product '{product.Name}' removed successfully.", MessageType.Success);
                }
            }
            else
            {
                ShowMessage("Product removal cancelled.", MessageType.Info);
            }
        }

        private void ViewLowStock()
        {
            Console.Clear();
            Console.WriteLine("LOW STOCK ALERT");
            Console.WriteLine(new string('-', 30));

            int threshold = GetValidatedInteger("Enter low stock threshold (default 5): ", 5);
            var lowStockProducts = inventory.GetLowStockProducts(threshold);

            if (!lowStockProducts.Any())
            {
                ShowMessage($"No products below {threshold} units in stock.", MessageType.Info);
                return;
            }

            Console.WriteLine($"Products with stock ≤ {threshold} units:\n");
            foreach (var product in lowStockProducts)
            {
                Console.WriteLine($"[ID: {product.Id}] {product.Name} - Stock: {product.StockQuantity} (Price: ${product.Price:F2})");
            }
        }

        private void CalculateTotalValue()
        {
            Console.Clear();
            Console.WriteLine("TOTAL INVENTORY VALUE");
            Console.WriteLine(new string('-', 30));

            decimal totalValue = inventory.CalculateTotalValue();
            var products = inventory.GetAllProducts();

            Console.WriteLine($"Number of Products: {products.Count}");
            Console.WriteLine($"Total Stock Items: {products.Sum(p => p.StockQuantity)}");
            Console.WriteLine($"Total Inventory Value: ${totalValue:F2}");
        }

        private void SearchProduct()
        {
            Console.Clear();
            Console.WriteLine("SEARCH PRODUCT");
            Console.WriteLine(new string('-', 30));

            Console.WriteLine("Search by:");
            Console.WriteLine("1. Product ID");
            Console.WriteLine("2. Product Name");
            
            string choice = GetUserInput("Choose search type (1-2): ");
            
            switch (choice)
            {
                case "1":
                    int id = GetValidatedInteger("Enter Product ID: ");
                    var product = inventory.FindProductById(id);
                    if (product != null)
                    {
                        Console.WriteLine("\nProduct Found:");
                        product.DisplayInfo();
                    }
                    else
                    {
                        ShowMessage($"Product with ID {id} not found.", MessageType.Error);
                    }
                    break;
                case "2":
                    string name = GetUserInput("Enter product name (or part of name): ");
                    var products = inventory.FindProductByName(name);
                    if (products.Any())
                    {
                        Console.WriteLine($"\nFound {products.Count} product(s):");
                        foreach (var p in products)
                        {
                            p.DisplayInfo();
                        }
                    }
                    else
                    {
                        ShowMessage($"No products found with name containing '{name}'.", MessageType.Info);
                    }
                    break;
                default:
                    ShowMessage("Invalid choice.", MessageType.Error);
                    break;
            }
        }

        // Helper methods for input validation
        private string GetUserInput(string prompt)
        {
            Console.Write(prompt);
            return Console.ReadLine()?.Trim() ?? "";
        }

        private string GetValidatedInput(string prompt, Func<string, bool> validator)
        {
            while (true)
            {
                string input = GetUserInput(prompt);
                if (validator(input))
                    return input;
                ShowMessage("Invalid input. Please try again.", MessageType.Error);
            }
        }

        private decimal GetValidatedDecimal(string prompt)
        {
            while (true)
            {
                string input = GetUserInput(prompt);
                if (decimal.TryParse(input, NumberStyles.Currency, CultureInfo.InvariantCulture, out decimal result) && result >= 0)
                    return result;
                ShowMessage("Please enter a valid positive number.", MessageType.Error);
            }
        }

        private int GetValidatedInteger(string prompt, int defaultValue = 0)
        {
            while (true)
            {
                string input = GetUserInput(prompt);
                if (string.IsNullOrWhiteSpace(input) && defaultValue != 0)
                    return defaultValue;
                if (int.TryParse(input, out int result) && result >= 0)
                    return result;
                ShowMessage("Please enter a valid positive integer.", MessageType.Error);
            }
        }

        private void ShowMessage(string message, MessageType type)
        {
            ConsoleColor originalColor = Console.ForegroundColor;
            
            switch (type)
            {
                case MessageType.Success:
                    Console.ForegroundColor = ConsoleColor.Green;
                    Console.WriteLine($"✓ {message}");
                    break;
                case MessageType.Error:
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine($"✗ {message}");
                    break;
                case MessageType.Info:
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine($"ℹ {message}");
                    break;
                case MessageType.Warning:
                    Console.ForegroundColor = ConsoleColor.Yellow;
                    Console.WriteLine($"⚠ {message}");
                    break;
            }

            Console.ForegroundColor = originalColor;
        }
    }

    // Enum for message types
    public enum MessageType
    {
        Success,
        Error,
        Info,
        Warning
    }
}
```

## 🚀 Getting Started
### Prerequisites
 - .NET SDK 5.0 or later
 - Visual Studio 2022+ or VS Code with C# extensions

### Installation & Running
#### 1. Clone/Create the Project
```
# Create new console project
dotnet new console -n InventoryManagementSystem
cd InventoryManagementSystem
```
#### 2. Replace Code
 - Copy the provided C# code into Program.cs
 - Or replace all files in the project with the provided classes
#### 3. Build and Run
```
# Build the application
dotnet build

# Run the application
dotnet run
```
#### 4. Alternative: Direct Compilation
```
csc *.cs
InventoryManagementSystem.exe
```

## 📖 Usage Guide
### Main Menu Options
```
1. Add New Product
2. View All Products
3. Update Stock
4. Remove Product
5. View Low Stock Alert
6. Calculate Total Inventory Value
7. Search Product
8. Save and Exit
```
### Detailed Feature Usage
#### 1. Adding Products
 - Enter product name, price, initial stock, and category
 - System assigns unique auto-incrementing ID
 - Data validation ensures correct input formats
### 2. Viewing Products
 - Displays all products in tabular format
 - Shows: ID, Name, Price, Stock, Category, Last Updated
 - Provides total count of products
### 3. Updating Stock
 - Three update methods:
   - Add stock (restocking)
   - Remove stock (sales)
   - Set exact quantity
 - Prevents negative stock quantities
### 4. Removing Products
 - Requires product ID
 - Confirmation prompt prevents accidental deletion
 - Immediate removal from inventory
### 5. Low Stock Alerts
 - Customizable threshold (default: 5 units)
 - Lists products below threshold
 - Helps with reordering decisions
### 6. Inventory Valuation
 - Calculates total value of all stock
 - Shows: Total products, Total items, Total value
 - Useful for financial reporting
### 7. Product Search
 - Search by Product ID (exact match)
 - Search by Product Name (partial match)
 - Displays all matching products
### 8. Data Persistence
 - Automatic save on exit
 - Manual save available
 - Data stored in inventory.json

## 🔧 Code Architecture
### Class Responsibilities
#### _Product_ Class
 - Represents individual inventory items
  - **Properties:** Id, Name, Price, StockQuantity, Category, LastUpdated
  - **Methods:** DisplayInfo(), UpdateStock()
#### _InventoryManager_ Class
 - Core business logic handler
 - Manages product collection
 - Methods for CRUD operations and queries
#### _FileManager_ Class
 - Handles JSON serialization/deserialization
 - Manages file I/O operations
 - Error handling for file operations
### _ConsoleUI_ Class
 - User interface controller
 - Menu navigation and input handling
 - Formatted output display
### _Program_ Class
 - Application entry point
 - Initializes and coordinates components

## Key Design Patterns
 - **Single Responsibility Principle:** Each class has a specific purpose
 - **Separation of Concerns:** UI, business logic, and data persistence are separate
 - **Repository Pattern:** InventoryManager acts as a repository for Product entities

## 📊 Data Storage
### File Format: JSON
```
[
  {
    "Id": 1,
    "Name": "Laptop",
    "Price": 999.99,
    "StockQuantity": 10,
    "Category": "Electronics",
    "LastUpdated": "2024-01-15T10:30:00"
  }
]
```

## Data Validation
 - **Price:** Positive decimal values
 - **Stock:** Non-negative integers
 - **Name/Category:** Non-empty strings
 - All inputs validated before processing

## 🎨 User Interface
### Color-Coded Messages
 - ✅ Green: Success messages
 - ❌ Red: Error messages
 - ℹ️ Cyan: Information messages
 - ⚠️ Yellow: Warning messages
### Input Validation
 - Type checking for numbers
 - Range validation for quantities
 - Confirmation for destructive actions

## 🧪 Testing
### Manual Testing Scenarios
 1. **Add Product:** Verify all fields are saved correctly
 2. **Update Stock:** Test positive and negative adjustments
 3. **Remove Product:** Confirm deletion and ID reassignment
 4. **Data Persistence:** Verify save/load functionality
 5. **Error Handling:** Test invalid inputs and edge cases
### Test Data Examples
```
// Sample test products
1. "Wireless Mouse", $29.99, 50, "Electronics"
2. "Office Chair", $199.99, 15, "Furniture"
3. "Coffee Mug", $9.99, 100, "Kitchenware"
```
