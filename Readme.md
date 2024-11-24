import java.io.*;
import java.util.*;

// Exception classes for custom error handling
class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) {
        super(message);
    }
}

class AccountNotFoundException extends Exception {
    public AccountNotFoundException(String message) {
        super(message);
    }
}

// Base class for User
class User {
    private String name;
    private String email;
    private String password;
    private List<Account> accounts;

    public User(String name, String email, String password) {
        this.name = name;
        this.email = email;
        this.password = password;
        this.accounts = new ArrayList<>();
    }

    // Getters and Setters
    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }

    public String getPassword() {
        return password;
    }

    public List<Account> getAccounts() {
        return accounts;
    }

    public void addAccount(Account account) {
        accounts.add(account);
    }

    public void saveAccountsToFile() {
        try (BufferedWriter writer = new BufferedWriter(new FileWriter(email + "_accounts.txt"))) {
            for (Account account : accounts) {
                writer.write(account.getAccountInfo());
                writer.newLine();
            }
        } catch (IOException e) {
            System.out.println("Error saving accounts: " + e.getMessage());
        }
    }

    public void loadAccountsFromFile() {
        try (BufferedReader reader = new BufferedReader(new FileReader(email + "_accounts.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) {
                String[] parts = line.split(",");
                String accountType = parts[0];
                String accountNumber = parts[1];
                double balance = Double.parseDouble(parts[2]);
                Account account;
                if ("Savings".equalsIgnoreCase(accountType)) {
                    double interestRate = Double.parseDouble(parts[3]);
                    account = new SavingsAccount(accountNumber, this, interestRate);
                } else {
                    double overdraftLimit = Double.parseDouble(parts[3]);
                    account = new CheckingAccount(accountNumber, this, overdraftLimit);
                }
                account.setBalance(balance);
                account.loadTransactionHistory();
                accounts.add(account);
            }
        } catch (IOException e) {
            System.out.println("Error loading accounts: " + e.getMessage());
        }
    }
}

// Abstract class for Account
abstract class Account {
    protected String accountNumber;
    protected double balance;
    protected User owner;
    protected List<String> transactionHistory;

    public Account(String accountNumber, User owner) {
        this.accountNumber = accountNumber;
        this.balance = 0.0; // Initial balance
        this.owner = owner;
        this.transactionHistory = new ArrayList<>();
    }

    public abstract void deposit(double amount);
    public abstract void withdraw(double amount) throws InsufficientFundsException;
    public abstract void displayAccountInfo();
    public abstract void transfer(Account targetAccount, double amount) throws InsufficientFundsException;

    public double getBalance() {
        return balance;
    }

    public void setBalance(double balance) {
        this.balance = balance;
    }

    public String getAccountInfo() {
        return this.getClass().getSimpleName() + "," + accountNumber + "," + balance;
    }

    protected void addTransaction(String transaction) {
        transactionHistory.add(transaction);
        saveTransactionHistory(); // Save transaction history after each transaction
    }

    public List<String> getTransactionHistory() {
        return transactionHistory;
    }

    public void saveTransactionHistory() {
        try (BufferedWriter writer = new BufferedWriter(new FileWriter(accountNumber + "_transactions.txt", true))) {
            for (String transaction : transactionHistory) {
                writer.write(transaction);
                writer.newLine();
            }
        } catch (IOException e) {
            System.out.println("Error saving transaction history: " + e.getMessage());
        }
    }

    public void loadTransactionHistory() {
        try (BufferedReader reader = new BufferedReader(new FileReader(accountNumber + "_transactions.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) {
                transactionHistory.add(line);
            }
        } catch (IOException e) {
            System.out.println("Error loading transaction history: " + e.get Message());
        }
    }

    // Notify user of transaction
    public void notifyUser (String message) {
        System.out.println("Notification for " + owner.getName() + ": " + message);
    }
}

// Savings Account class
class SavingsAccount extends Account {
    private double interestRate;

    public SavingsAccount(String accountNumber, User owner, double interestRate) {
        super(accountNumber, owner);
        this.interestRate = interestRate;
    }

    @Override
    public void deposit(double amount) {
        balance += amount;
        addTransaction("Deposited " + amount);
        notifyUser ("Deposited " + amount + " to Savings Account.");
    }

    @Override
    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount <= balance) {
            balance -= amount;
            addTransaction("Withdrew " + amount);
            notifyUser ("Withdrew " + amount + " from Savings Account.");
        } else {
            throw new InsufficientFundsException("Insufficient funds for withdrawal!");
        }
    }

    @Override
    public void displayAccountInfo() {
        System.out.println("Savings Account Number: " + accountNumber);
        System.out.println("Balance: " + balance);
        System.out.println("Interest Rate: " + interestRate);
    }

    @Override
    public void transfer(Account targetAccount, double amount) throws InsufficientFundsException {
        withdraw(amount);
        targetAccount.deposit(amount);
        addTransaction("Transferred " + amount + " to " + targetAccount.accountNumber);
        notifyUser ("Transferred " + amount + " to " + targetAccount.accountNumber);
    }
}

// Checking Account class
class CheckingAccount extends Account {
    private double overdraftLimit;

    public CheckingAccount(String accountNumber, User owner, double overdraftLimit) {
        super(accountNumber, owner);
        this.overdraftLimit = overdraftLimit;
    }

    @Override
    public void deposit(double amount) {
        balance += amount;
        addTransaction("Deposited " + amount);
        notifyUser ("Deposited " + amount + " to Checking Account.");
    }

    @Override
    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount <= balance + overdraftLimit) {
            balance -= amount;
            addTransaction("Withdrew " + amount);
            notifyUser ("Withdrew " + amount + " from Checking Account.");
        } else {
            throw new InsufficientFundsException("Overdraft limit exceeded!");
        }
    }

    @Override
    public void displayAccountInfo() {
        System.out.println("Checking Account Number: " + accountNumber);
        System.out.println("Balance: " + balance);
        System.out.println("Overdraft Limit: " + overdraftLimit);
    }

    @Override
    public void transfer(Account targetAccount, double amount) throws InsufficientFundsException {
        withdraw(amount);
        targetAccount.deposit(amount);
        addTransaction("Transferred " + amount + " to " + targetAccount.accountNumber);
        notifyUser ("Transferred " + amount + " to " + targetAccount.accountNumber);
    }
}

// Banking System class for user management and operations
class BankingSystem {
    private Map<String, User> users;
    private final String dataFile = "banking_data.txt";

    public BankingSystem() {
        users = new HashMap<>();
        loadUsers();
    }

    public void registerUser (String name, String email, String password) {
        if (!users.containsKey(email)) {
            User newUser  = new User(name, email, password);
            users.put(email, newUser );
            saveUser (newUser );
            System.out.println("User  registered successfully.");
        } else {
            System.out.println("User  already exists.");
        }
    }

    public User loginUser (String email, String password) {
        User user = users.get(email);
        if (user != null && user.getPassword().equals(password)) {
            System.out.println("Login successful.");
            return user;
        } else {
            System.out.println("Invalid email or password.");
            return null;
        }
    }

    public void displayAllUsers() {
        for (User  user : users.values()) {
            System.out.println(":User  " + user.getName() + ", Email: " + user.getEmail());
        }
    }

    private void saveUser (User user) {
        try (BufferedWriter writer = new BufferedWriter(new FileWriter(dataFile, true))) {
            writer.write(user.getName() + "," + user.getEmail() + "," + user.getPassword());
            writer.newLine();
        } catch (IOException e) {
            System.out.println("Error saving user data: " + e.getMessage());
        }
    }

    private void loadUsers() {
        try (BufferedReader reader = new BufferedReader(new FileReader(dataFile))) {
            String line;
            while (( line = reader.readLine()) != null) {
                String[] parts = line.split(",");
                if (parts.length == 3) {
                    User user = new User(parts[0], parts[1], parts[2]);
                    user.loadAccountsFromFile();
                    users.put(parts[1], user);
                }
            }
        } catch (IOException e) {
            System.out.println("Error loading user data: " + e.getMessage());
        }
    }
}

// Main class to run the advanced banking application
public class AdvancedBankingApp {
    public static void main(String[] args) {
        BankingSystem bankingSystem = new BankingSystem();
        Scanner scanner = new Scanner(System.in);
        int choice;

        do {
            System.out.println("1. Register User");
            System.out.println("2. Login User");
            System.out.println("3. Display All Users");
            System.out.println("4. Exit");
            System.out.print("Enter your choice: ");
            choice = scanner.nextInt();
            scanner.nextLine(); // Consume newline

            switch (choice) {
                case 1:
                    System.out.print("Enter name: ");
                    String name = scanner.nextLine();
                    System.out.print("Enter email: ");
                    String email = scanner.nextLine();
                    System.out.print("Enter password: ");
                    String password = scanner.nextLine();
                    bankingSystem.registerUser (name, email, password);
                    break;

                case 2:
                    System.out.print("Enter email: ");
                    String loginEmail = scanner.nextLine();
                    System.out.print("Enter password: ");
                    String loginPassword = scanner.nextLine();
                    User loggedInUser  = bankingSystem.loginUser (loginEmail, loginPassword);

                    if (loggedInUser  != null) {
                        int userChoice;
                        do {
                            System.out.println("1. Create Savings Account");
                            System.out.println("2. Create Checking Account");
                            System.out.println("3. View Accounts");
                            System.out.println("4. View Transaction History");
                            System.out.println("5. Exit to Main Menu");
                            System.out.print("Enter your choice: ");
                            userChoice = scanner.nextInt();

                            switch (userChoice) {
                                case 1:
                                    System.out.print("Enter account number: ");
                                    String savingsAccountNumber = scanner.next();
                                    System.out.print("Enter interest rate: ");
                                    double interestRate = scanner.nextDouble();
                                    SavingsAccount savingsAccount = new SavingsAccount(savingsAccountNumber, loggedInUser , interestRate);
                                    loggedInUser .addAccount(savingsAccount);
                                    loggedInUser .saveAccountsToFile();
                                    System.out.println("Savings Account created successfully.");
                                    break;

                                case 2:
                                    System.out.print("Enter account number: ");
                                    String checkingAccountNumber = scanner.next();
                                    System.out.print("Enter overdraft limit: ");
                                    double overdraftLimit = scanner.nextDouble();
                                    CheckingAccount checkingAccount = new CheckingAccount(checkingAccountNumber, loggedInUser , overdraftLimit);
                                    loggedInUser .addAccount(checkingAccount);
                                    loggedInUser .saveAccountsToFile();
                                    System.out.println("Checking Account created successfully.");
                                    break;

                                case 3:
                                    System.out.println("Accounts for " + loggedInUser .getName() + ":");
                                    for (Account account : loggedInUser .getAccounts()) {
                                        account.displayAccountInfo();
                                    }
                                    break;

                                case 4:
                                    System.out.println("Transaction History for Accounts:");
                                    for (Account account : loggedInUser .getAccounts()) {
                                        System.out.println("Transaction History for Account Number: " + account.accountNumber);
                                        List<String> transactions = account.getTransactionHistory();
                                        if (transactions.isEmpty()) {
                                            System.out.println("No transactions found.");
                                        } else {
                                            for (String transaction : transactions) {
                                                System.out.println(transaction);
                                            }
                                        }
                                    }
                                    break;

                                case 5:
                                    System.out.println("Exiting to Main Menu.");
                                    break;

                                default:
                                    System.out.println("Invalid choice. Please try again.");
                            }
                        } while (userChoice != 5);
                    }
                    break;

                case 3:
                    bankingSystem.displayAllUsers();
                    break;

                case 4:
                    System.out.println("Exiting the application.");
                    break;

                default:
                    System.out.println("Invalid choice. Please try again.");
            }
        } while (choice != 4);

        scanner.close();
    }
}