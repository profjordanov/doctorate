MediatR Pipeline Examples
13 October, 2016. It was a Thursday.
A while ago, I blogged about using MediatR to build a processing pipeline for requests in the form of commands and queries in your application. MediatR is a library I built (well, extracted from client projects) to help organize my architecture into a CQRS architecture with distinct messages and handlers for every request in your system.

So when processing requests gets more complicated, we often rely on a mediator pipeline to provide a means for these extra behaviors. It doesn’t always show up – I’ll start without one before deciding to add it. I’ve also not built it in directly to MediatR  – because frankly, it’s hard and there are existing tools to do so with modern DI containers. First, let’s look at the simplest pipeline that could possible work:

public class MediatorPipeline<TRequest, TResponse> 
  : IRequestHandler<TRequest, TResponse>
  where TRequest : IRequest<TResponse>
{
    private readonly IRequestHandler<TRequest, TResponse> _inner;

    public MediatorPipeline(IRequestHandler<TRequest, TResponse> inner)
    {
        _inner = inner;
    }

    public TResponse Handle(TRequest message)
    {
        return _inner.Handle(message);
    }
 }
view rawBlankPipeline.cs hosted with ❤ by GitHub
Nothing exciting here, it just calls the inner handler, the real handler. But we have a baseline that we can layer on additional behaviors.

Let’s get something more interesting going!

Contextual Logging and Metrics
Serilog has an interesting feature where it lets you define contexts for logging blocks. With a pipeline, this becomes trivial to add to our application:

public class MediatorPipeline<TRequest, TResponse> 
  : IRequestHandler<TRequest, TResponse>
  where TRequest : IRequest<TResponse>
{
    private readonly IRequestHandler<TRequest, TResponse> _inner;

    public MediatorPipeline(IRequestHandler<TRequest, TResponse> inner)
    {
        _inner = inner;
    }

    public TResponse Handle(TRequest message)
    {
        using (LogContext.PushProperty(LogConstants.MediatRRequestType, typeof(TRequest).FullName))
        {
            return _inner.Handle(message);
        }
    }
 }
view rawSerilogPipeline.cs hosted with ❤ by GitHub
In our logs, we’ll now see a logging block right before we enter our handler, and right after we exit. We can do a bit more, what about metrics? Also trivial to add:

using (LogContext.PushProperty(LogConstants.MediatRRequestType, requestType))
using (Metrics.Time(Timers.MediatRRequest))
{
    return _inner.Handle(request);
}
view rawMetricsPipeline.cs hosted with ❤ by GitHub
That Time class is just a simple wrapper around the .NET Timer classes, with some configuration checking etc. Those are the easy ones, what about something more interesting?

Validation and Authorization
Often times, we have to share handlers between different applications, so it’s important to have an agnostic means of cross-cutting concerns. Rather than bury our concerns in framework or application-specific extensions (like, say, an action filter), we can instead embed this behavior in our pipeline. First, with validation, we can use a tool like Fluent Validation with validator handlers for a specific type:

public interface IMessageValidator<in T>
{
    IEnumerable<ValidationFailure> Validate(T message);
}
view rawMessageValidator.cs hosted with ❤ by GitHub
What’s interesting here is that our message validator is contravariant, meaning I can have a validator of a base type work for messages of a derived type. That means we can declare common validators for base types or interfaces that your message inherits/implements. In practice this lets me share common validation amongst multiple messages simply by implementing an interface.

Inside my pipeline, I can execute my validation my taking a dependency on the validators for my message:

public class MediatorPipeline<TRequest, TResponse> 
  : IRequestHandler<TRequest, TResponse>
  where TRequest : IRequest<TResponse>
{
    private readonly IRequestHandler<TRequest, TResponse> _inner;
    private readonly IEnumearble<IMessageValidator<TRequest>> _validators;

    public MediatorPipeline(IRequestHandler<TRequest, TResponse> inner,
        IEnumerable<IMessageValidator<TRequest>> validators)
    {
        _inner = inner;
        _validators = validators;
    }

    public TResponse Handle(TRequest message)
    {
        using (LogContext.PushProperty(LogConstants.MediatRRequestType, requestType))
        using (Metrics.Time(Timers.MediatRRequest))
        {
            var failuers = _validators
                .Select(v => v.Validate(message))
                .SelectMany(result => result.Errors)
                .Where(f => f != null)
                .ToList();
            if (failures.Any())
                throw new ValidationException(failures);
            
            return _inner.Handle(request);
        }
    }
 }
view rawValidatorPipeline.cs hosted with ❤ by GitHub
And bundle up all my errors into a potential exception thrown. The downside of this approach is I’m using exceptions to provide control flow, so if this is a problem, I can wrap up my responses into some sort of Result object that contains potential validation failures. In practice it seems fine for the applications we build.

Again, my calling code INTO my handler (the Mediator) has no knowledge of this new behaviors, nor does my handler. I go to one spot to augment and extend behaviors across my entire system. Keep in mind, however, I still place my validators beside my message, handler, view etc. using feature folders.

Authorization is similar, where I define an authorizer of a message:

public interface IMessageAuthorizer {
  void Evaluate<TRequest>(TRequest request) where TRequest : class
}
view rawIMessageAuthorizer.cs hosted with ❤ by GitHub
Then in my pipeline, check authorization:

public class MediatorPipeline<TRequest, TResponse> 
  : IRequestHandler<TRequest, TResponse>
  where TRequest : IRequest<TResponse>
{
    private readonly IRequestHandler<TRequest, TResponse> _inner;
    private readonly IEnumearble<IMessageValidator<TRequest>> _validators;
    private readonly IMessageAuthorizer _authorizer;

    public MediatorPipeline(IRequestHandler<TRequest, TResponse> inner,
        IEnumerable<IMessageValidator<TRequest>> validator,
        IMessageAuthorizor authorizer
        )
    {
        _inner = inner;
        _validators = validators;
        _authorizer = authorizer;
    }

    public TResponse Handle(TRequest message)
    {
        using (LogContext.PushProperty(LogConstants.MediatRRequestType, requestType))
        using (Metrics.Time(Timers.MediatRRequest))
        {
            _securityHandler.Evaluate(message);
            
            var failures = _validators
                .Select(v => v.Validate(message))
                .SelectMany(result => result.Errors)
                .Where(f => f != null)
                .ToList();
            if (failures.Any())
                throw new ValidationException(failures);
            
            return _inner.Handle(request);
        }
    }
 }
view rawAuthorizingPipeline.cs hosted with ❤ by GitHub
The actual implementation of the authorizer will go through a series of security rules, find matching rules, and evaluate them against my request. Some examples of security rules might be:

Do any of your roles have permission?
Are you part of the ownership team of this resource?
Are you assigned to a special group that this resource is associated with?
Do you have the correct training to perform this action?
Are you in the correct geographic location and/or citizenship?
Things can get pretty complicated, but again, all encapsulated for me inside my pipeline.

Finally, what about potential augmentations or reactions to a request?

Pre/post processing
In addition to some specific processing needs, like logging, metrics, authorization, and validation, there are things I can’t predict one message or group of messages might need. For those, I can build some generic extension points:

public interface IPreRequestHandler<in TRequest>
{
    void Handle(TRequest);
}
public interface IPostRequestHandler<in TRequest, in TResponse>
{
    void Handle(TRequest request, TResponse response);
}
public interface IResponseHandler<in TResponse>
{
    void Handle(TResponse response);
}
view rawRequestProcessors.cs hosted with ❤ by GitHub
Next I update my pipeline to include calls to these extensions (if they exist):

public class MediatorPipeline<TRequest, TResponse> 
  : IRequestHandler<TRequest, TResponse>
  where TRequest : IRequest<TResponse>
{
    private readonly IRequestHandler<TRequest, TResponse> _inner;
    private readonly IEnumearble<IMessageValidator<TRequest>> _validators;
    private readonly IMessageAuthorizer _authorizer;
    private readonly IEnumerable<IPreRequestProcessor<TRequest>> _preProcessors;
    private readonly IEnumerable<IPostRequestProcessor<TRequest, TResponse>> _postProcessors;
    private readonly IEnumerable<IResponseProcessor<TResponse>> _responseProcessors;

    public MediatorPipeline(IRequestHandler<TRequest, TResponse> inner,
        IEnumerable<IMessageValidator<TRequest>> validator,
        IMessageAuthorizor authorizer,
        IEnumerable<IPreRequestProcessor<TRequest>> preProcessors,
        IEnumerable<IPostRequestProcessor<TRequest, TResponse>> postProcessors,
        IEnumerable<IResponseProcessor<TResponse>> responseProcessors
        )
    {
        _inner = inner;
        _validators = validators;
        _authorizer = authorizer;
        _preProcessors = preProcessors;
        _postProcessors = postProcessors;
        _responseProcessors = responseProcessors;
    }

    public TResponse Handle(TRequest message)
    {
        using (LogContext.PushProperty(LogConstants.MediatRRequestType, requestType))
        using (Metrics.Time(Timers.MediatRRequest))
        {
            _securityHandler.Evaluate(message);
            
            foreach (var preProcessor in _preProcessors)
                preProcessor.Handle(request);
            
            var failures = _validators
                .Select(v => v.Validate(message))
                .SelectMany(result => result.Errors)
                .Where(f => f != null)
                .ToList();
            if (failures.Any())
                throw new ValidationException(failures);
            
            var response = _inner.Handle(request);
            
            foreach (var postProcessor in _postProcessors)
                postProcessor.Handle(request, response);
                
            foreach (var responseProcessor in _responseProcessors)
                responseProcessor.Handle(response);
                
            return response;
        }
    }
 }
view rawProcessingPipeline.cs hosted with ❤ by GitHub
So what kinds of things might I accomplish here?

Supplementing my request with additional information not to be found in the original request (in one case, barcode sequences)
Data cleansing or fixing (for example, a scanned barcode needs padded zeroes)
Limiting results of paged result models via configuration
Notifications based on the response
All sorts of things that I could put inside the handlers, but if I want to apply a general policy across many handlers, can quite easily be accomplished.

Whether you have specific or generic needs, a mediator pipeline can be a great place to apply domain-centric behaviors to all requests, or only matching requests based on generics rules, across your entire application.



What is CQRS ?
In traditional architecture the same data model is used for both read and write operations. By using CQRS — Command and Query Responsibility Segregation we can separate write operations (commands) and the read operations (queries) into separate models in an application.

The primary objectives of CQRS is to improve the scalability and performance of an application by separating the read and write operations into different models, each optimized for their specific task. This can lead to a more flexible and maintainable application architecture, especially in enterprise level large-scale applications with high read and write loads.


What is MediatR?
MediatR is a NuGet package for .NET applications that helps to implement the Mediator pattern and MediatR can be used as a useful tool to implement CQRS in a .NET application.

MediatR pattern helps to reduce direct dependency between multiple objects and make them collaborative through MediatR.

Demo Time!
Let’s install MediatR first. To use MediatR in a .NET Core application, you need to add the MediatR NuGet package to your project. You can do this using the following command in the Package Manager Console

Install-Package MediatR
Once you have added the MediatR package, you need to add the MediatR middleware to your application’s Startup.cs. You can do this by adding the following code to your ConfigureServices method:

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {        
        services.AddMediatR(typeof(Startup));        
    }
}
Now let’s get a real world scenario and apply MediatR pattern to the application. Let’s take Customer- Order sample scenario and implement CQRS using MediatR.

Let’s define the order entity that we’ll be working with,

public class Order
{
    public int Id { get; set; }
    public string CustomerName { get; set; }
    public DateTime OrderDate { get; set; }
}
Next, let’s define the commands and queries that we’ll need for the system. We’ll need commands to create, update, and cancel orders, as well as a query to retrieve a list of all orders:

public class CreateOrderCommand : IRequest<int>
{
    public string CustomerName { get; set; }
    public DateTime OrderDate { get; set; }
}

public class UpdateOrderCommand : IRequest<Order>
{
    public int OrderId { get; set; }
    public string CustomerName { get; set; }
    public DateTime OrderDate { get; set; }
}

public class DeleteOrderCommand : IRequest<Order>
{
    public int OrderId { get; set; }
}

public class GetOrderListQuery : IRequest<List<Order>>
{
    public int OrderId { get; set; }
}
In this example, the CreateOrderCommand request represents a command to create a new order for a customer with a given ID. The GetOrderQuery request represents a query to retrieve the details of an order with a given ID.

Next, define a handler for each request:

public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IOrderRepository _orderRepository;

    public CreateOrderCommandHandler(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public Task<int> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        var order = new Order
        {
            Id = _orders.Count + 1,
            CustomerName = request.CustomerName,
            OrderDate = request.OrderDate
        };

        var orderId = await _orderRepository.Create(order);       

        return Task.FromResult(orderId);
    }
}
Sample code of CreateOrderHandler looks like above,

In this example, the CreateOrderHandler creates a new order in the system by creating a new Order object and storing it in the database via an IOrderRepository.

public class UpdateOrderCommandHandler : IRequestHandler<UpdateOrderCommand, Unit>
{
    private readonly IOrderRepository _orderRepository;

    public UpdateOrderCommandHandler(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public Task<Order> Handle(UpdateOrderCommand request, CancellationToken cancellationToken)
    {
        var order = _orderRepository.FirstOrDefault(o => o.Id == request.OrderId);

        order.CustomerName = request.CustomerName;
        order.OrderDate = request.OrderDate;
        
        var orderUpdated = await _orderRepository.Update(order);
        return Task.FromResult(orderUpdated);
    }
}
Sample code of UpdateOrderCommandHandler looks like above,

The GetOrderQuery request is sent to the GetOrderHandler, which retrieves the details of the order with the given ID.

public class GetOrderHandler : IRequestHandler<GetOrderQuery, Order>
{
    private readonly IOrderRepository _orderRepository;

    public GetOrderHandler(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }

    public async Task<Order> Handle(GetOrderQuery request, CancellationToken cancellationToken)
    {
        var order = await _orderRepository.GetById(request.OrderId);
        return order;
    }
}
Finally, you can use MediatR to send requests to the appropriate handler:

var mediator = new Mediator(serviceProvider);

var createOrderCommand = new CreateOrderCommand
{
    CustomerId = 1,
    CustomerName = 'Test Customer'
};

var orderId = await mediator.Send(createOrderCommand);
While Mediator has many advantages, there are also some disadvantages to consider:

Debugging can be challenging: Because the mediator pattern introduces an additional layer of abstraction between commands and their handlers, it can be more difficult to trace and debug issues in your code.
Potential for circular dependencies: If you’re not careful, you can introduce circular dependencies between commands and their handlers, which can cause issues in your code.
It’s important to weigh the advantages and disadvantages of Mediator carefully before deciding to use it in your project. While it can be a powerful tool for managing communication between objects in your system, it’s not always the best choice for every situation.


..............

https://code-maze.com/cqrs-mediatr-in-aspnet-core/