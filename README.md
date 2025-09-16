## ApiGateway
<img width="1024" height="500" alt="Black and White Simple Square and Calligraphy Business Logo" src="https://github.com/user-attachments/assets/e9c6aa4c-28f1-4833-b042-01471a18c05c" />

## ERD
<img width="641" height="473" alt="Screenshot 2025-09-16 at 5 12 19 PM" src="https://github.com/user-attachments/assets/29f12b9b-8c09-4d1f-b108-c9e811e5a9b3" />

## Project Overview  

This project follows a **Microservices Architecture**, with each service responsible for a specific domain.  

### Key Features  
- **Each service has its own SQL schema** .  
- **Database migration** support for version control.  
- **API Gateway** using Spring Cloud with OpenFeign for service-to-service communication.  
- **JWT Authentication** with **Role-Based Authorization**.  
- **Member management** with full CRUD support.  
- **Race condition handling** in critical transactions.  
- **Rate Limiting** to protect APIs from abuse.  
- **API Documentation** using Swagger/OpenAPI.  
- **Member Activity Logs** for tracking actions.  
- **Pagination** .  

### Microservices  
1. **Books Service**  
   - Handles **Authors**, **Publishers**, and **Books**.  
2. **Borrowing Transactions Service**  
   - Manages everything related to borrowing, returning, and tracking books.  
3. **User Service**  
   - Responsible for **Role-Based Authentication**, **Authorization**, and **Member Routing**.  

### API Documentation  
- Postman Collection: [Click here to view](https://postman.co/workspace/My-Workspace~ea8525f4-631b-48a4-8869-ffae1f0aa998/collection/32005719-32c7a4b2-321b-400b-b3ff-1fbf0421a14e?action=share&creator=32005719](https://www.postman.com/mohamedgamal167/workspace/code-81/collection/32005719-32c7a4b2-321b-400b-b3ff-1fbf0421a14e?action=share&creator=32005719))  

### Running the Project  
Make sure to run the following command inside one of the services before starting the application:  

```bash
docker compose up -d

 
