package main

import (
	"bufio"
	"fmt"
	"os"
	"os/exec"
	"strings"
	"time"

	tgbotapi "gopkg.in/telegram-bot-api.v4"
)

const (
	subdomainsFile = "subdomains.txt"
	token          = "6561974676:AAGi6vWMBNmjnHpK70Zt0_rcW2A8WYZ-iAQ"
	chatID         = 6561974676
)

func main() {
	// Get target domain from user input
	reader := bufio.NewReader(os.Stdin)
	fmt.Print("Enter target domain (e.g., example.com): ")
	domain, _ := reader.ReadString('\n')
	domain = strings.TrimSpace(domain)

	// Run recon every 12 hours
	ticker := time.NewTicker(10 * time.Minute)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			fmt.Println("Starting recon...")
			runRecon(domain)
			notifyTelegram()
		}
	}
}

func runRecon(domain string) {
	// Execute subfinder
	cmd := exec.Command("subfinder", "-d", domain, "-o", subdomainsFile)
	_, err := cmd.Output()
	if err != nil {
		fmt.Println("Error executing subfinder:", err)
		return
	}
	fmt.Println("Subdomain enumeration complete.")
}

func notifyTelegram() {
	// Read new subdomains
	newSubdomains := readSubdomainsFromFile(subdomainsFile)
	if len(newSubdomains) == 0 {
		fmt.Println("No new subdomains found.")
		return
	}

	// Read existing subdomains
	existingSubdomains := readSubdomainsFromFile(subdomainsFile)

	// Filter out already existing subdomains
	var filteredSubdomains []string
	for _, subdomain := range newSubdomains {
		if !contains(existingSubdomains, subdomain) {
			filteredSubdomains = append(filteredSubdomains, subdomain)
		}
	}

	// Notify on Telegram
	bot, err := tgbotapi.NewBotAPI(token)
	if err != nil {
		fmt.Println("Error initializing Telegram bot:", err)
		return
	}

	msg := tgbotapi.NewMessage(chatID, "New subdomains found:\n"+strings.Join(filteredSubdomains, "\n"))
	_, err = bot.Send(msg)
	if err != nil {
		fmt.Println("Error sending message:", err)
		return
	}

	fmt.Println("Notification sent to Telegram.")
}

func readSubdomainsFromFile(filename string) []string {
	file, err := os.Open(filename)
	if err != nil {
		fmt.Println("Error opening file:", err)
		return nil
	}
	defer file.Close()

	var subdomains []string
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		subdomains = append(subdomains, scanner.Text())
	}
	if err := scanner.Err(); err != nil {
		fmt.Println("Error scanning file:", err)
		return nil
	}

	return subdomains
}

func contains(slice []string, item string) bool {
	for _, elem := range slice {
		if elem == item {
			return true
		}
	}
	return false
}
