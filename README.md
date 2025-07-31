import json
from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

class GCPAdminClient:
    def __init__(self, service_account_file, domain, admin_email):
        """
        Initialize the GCP Admin SDK client
        
        Args:
            service_account_file (str): Path to the service account JSON file
            domain (str): Your Google Workspace domain
            admin_email (str): Email of an admin user to impersonate
        """
        self.domain = domain
        self.admin_email = admin_email
        
        # Define the scopes required for Admin SDK
        scopes = [
            'https://www.googleapis.com/auth/admin.directory.user',
            'https://www.googleapis.com/auth/admin.directory.user.readonly'
        ]
        
        # Load service account credentials
        credentials = service_account.Credentials.from_service_account_file(
            service_account_file, scopes=scopes
        )
        
        # Impersonate an admin user (required for Admin SDK)
        self.delegated_credentials = credentials.with_subject(admin_email)
        
        # Build the Admin SDK service
        self.service = build('admin', 'directory_v1', credentials=self.delegated_credentials)

    def get_all_users(self, max_results=500, include_suspended=True):
        """
        Get all users from the domain
        
        Args:
            max_results (int): Maximum number of users to return per page
            include_suspended (bool): Include suspended users in results
            
        Returns:
            list: List of user objects
        """
        users = []
        page_token = None
        
        try:
            while True:
                # Build the request parameters
                params = {
                    'domain': self.domain,
                    'maxResults': max_results,
                    'orderBy': 'email'
                }
                
                if page_token:
                    params['pageToken'] = page_token
                
                if not include_suspended:
                    params['query'] = 'isSuspended=false'
                
                # Execute the request
                result = self.service.users().list(**params).execute()
                
                # Add users to our list
                if 'users' in result:
                    users.extend(result['users'])
                
                # Check if there are more pages
                page_token = result.get('nextPageToken')
                if not page_token:
                    break
                    
        except HttpError as error:
            print(f'An error occurred: {error}')
            return []
        
        return users

    def get_user_by_email(self, email):
        """
        Get a specific user by email address
        
        Args:
            email (str): User's email address
            
        Returns:
            dict: User object or None if not found
        """
        try:
            user = self.service.users().get(userKey=email).execute()
            return user
        except HttpError as error:
            print(f'An error occurred: {error}')
            return None

    def search_users(self, query, max_results=100):
        """
        Search for users based on a query
        
        Args:
            query (str): Search query (e.g., "name:John" or "orgUnitPath='/Marketing'")
            max_results (int): Maximum number of results to return
            
        Returns:
            list: List of matching user objects
        """
        try:
            result = self.service.users().list(
                domain=self.domain,
                query=query,
                maxResults=max_results,
                orderBy='email'
            ).execute()
            
            return result.get('users', [])
            
        except HttpError as error:
            print(f'An error occurred: {error}')
            return []

    def print_user_summary(self, users):
        """
        Print a summary of users
        
        Args:
            users (list): List of user objects
        """
        print(f"\nTotal users found: {len(users)}")
        print("-" * 80)
        
        for user in users:
            name = user.get('name', {})
            full_name = name.get('fullName', 'N/A')
            email = user.get('primaryEmail', 'N/A')
            suspended = user.get('suspended', False)
            org_unit = user.get('orgUnitPath', 'N/A')
            last_login = user.get('lastLoginTime', 'Never')
            
            status = "SUSPENDED" if suspended else "ACTIVE"
            
            print(f"Name: {full_name}")
            print(f"Email: {email}")
            print(f"Status: {status}")
            print(f"Org Unit: {org_unit}")
            print(f"Last Login: {last_login}")
            print("-" * 40)


def main():
    # Configuration
    SERVICE_ACCOUNT_FILE = 'path/to/your/service-account-key.json'
    DOMAIN = 'yourdomain.com'
    ADMIN_EMAIL = 'admin@yourdomain.com'  # Must be a super admin
    
    try:
        # Initialize the client
        admin_client = GCPAdminClient(SERVICE_ACCOUNT_FILE, DOMAIN, ADMIN_EMAIL)
        
        # Get all users
        print("Fetching all users...")
        all_users = admin_client.get_all_users()
        admin_client.print_user_summary(all_users)
        
        # Example: Get a specific user
        print("\n" + "="*80)
        print("Getting specific user...")
        specific_user = admin_client.get_user_by_email('user@yourdomain.com')
        if specific_user:
            admin_client.print_user_summary([specific_user])
        
        # Example: Search for users
        print("\n" + "="*80)
        print("Searching for users in Marketing org unit...")
        marketing_users = admin_client.search_users("orgUnitPath='/Marketing'")
        admin_client.print_user_summary(marketing_users)
        
        # Example: Get only active users
        print("\n" + "="*80)
        print("Getting only active users...")
        active_users = admin_client.get_all_users(include_suspended=False)
        admin_client.print_user_summary(active_users)
        
    except Exception as e:
        print(f"Error initializing Admin SDK client: {e}")


if __name__ == '__main__':
    main()