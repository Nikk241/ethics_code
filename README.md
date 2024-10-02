# ethics_code
import pandas as pd
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options

# Load the categories and websites from CSV file
categories_df = pd.read_csv('site_list.csv')

# Convert the DataFrame to a dictionary
categories = categories_df.groupby('category')['website'].apply(list).to_dict()

# Set up Chrome WebDriver options
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run Chrome in headless mode (no GUI)

# Initialize the WebDriver using ChromeDriverManager and options
driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=chrome_options)

cookies_data = {category: [] for category in categories}
failed_sites = []  # List to store sites that failed to load

try:
    for category, sites in categories.items():
        for website in sites:
            try:
                driver.get(website)
                
                # Get all cookies
                cookies = driver.get_cookies()
                
                # Store cookies in the list
                for cookie in cookies:
                    # Ensure domain starts with 'www.' if it doesn't
                    if cookie['domain'].startswith('.'):
                        cookie['domain'] = 'www' + cookie['domain']
                    elif not cookie['domain'].startswith('www.'):
                        cookie['domain'] = 'www.' + cookie['domain']
                    
                    cookie['website'] = website  # Add the website URL to each cookie dictionary
                    cookies_data[category].append(cookie)
            
            except Exception as e:
                print(f"Exception occurred while fetching cookies from {website}: {str(e)}")
                failed_sites.append(website)
            
except Exception as e:
    print(f"Exception occurred: {str(e)}")

finally:
    driver.quit()

# Convert each category's list of cookies to a DataFrame
dfs = {category: pd.DataFrame(cookies) for category, cookies in cookies_data.items()}

# Combine all DataFrames into one, filling missing values with NaN
cookies_df = pd.concat(dfs.values(), keys=dfs.keys(), names=['Category']).reset_index(level=0)

# Save the DataFrame to a CSV file
cookies_df.to_csv('website_cookies.csv', index=False)

# Print the DataFrame
print(cookies_df)

# Print failed sites for troubleshooting
print("\nFailed to fetch cookies from the following sites:")
for site in failed_sites:
    print(site)
