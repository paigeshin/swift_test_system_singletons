# swift_test_system_singletons

```swift
/// STEPS TO TEST SYSTEM SINGLETONS
/*
 
 1. Abstract into a protocol

 2. Use the protocol with the singleton as the default

 3. Mock the protocol in your tests
 
 */

/// Code that uses Singleton `URLSession.shard`
/*
 This DataLoader is currently very hard to test, as it will automatically call the shared URL session and perform a network call. This would require us to add waiting and timeouts to our testing code, and it quickly becomes very tricky and unstable.
 */

class DataLoader {
    enum Result {
        case data(Data)
        case error(Error)
    }

    func load(from url: URL, completionHandler: @escaping (Result) -> Void) {
        let task: URLSessionDataTask = URLSession.shared.dataTask(with: url) { (data, response, error) in
            if let error = error {
                return completionHandler(.error(error))
            }

            completionHandler(.data(data ?? Data()))
        }

        task.resume()
    }
}


// MARK:  1. Abstract into a protocol
protocol NetworkEngine {
    typealias Handler = (Data?, URLResponse?, Error?) -> Void

    func performRequest(for url: URL, completionHandler: @escaping Handler)
}

extension URLSession: NetworkEngine {
    typealias Handler = NetworkEngine.Handler

    func performRequest(for url: URL, completionHandler: @escaping Handler) {
        let task: URLSessionDataTask = self.dataTask(with: url, completionHandler: completionHandler)
        task.resume()
    }
}

// MARK: 2. Use the protocol with the singleton as the default
class DataLoader {
    
    enum Result {
        case data(Data)
        case error(Error)
    }
    
    private let engine: NetworkEngine
    
    init(engine: NetworkEngine = URLSession.shared) {
        self.engine = engine
    }
    
    func load(from url: URL, completionHandler: @escaping (Result) -> Void) {
        self.engine.performRequest(for: url) { (data, response, error) in
            if let error = error {
                return completionHandler(.error(error))
            }

            completionHandler(.data(data ?? Data()))
        }
    }
    
}

// MARK: 3. Mock the protocol in your tests
func testLoadingData() {
    class NetworkEngineMock: NetworkEngine {
        typealias Handler = NetworkEngine.Handler

        var requestedURL: URL?

        func performRequest(for url: URL, completionHandler: @escaping Handler) {
            self.requestedURL = url

            let data = "Hello world".data(using: .utf8)
            completionHandler(data, nil, nil)
        }
    }

    let engine = NetworkEngineMock()
    let loader = DataLoader(engine: engine)

    var result: DataLoader.Result?
    let url = URL(string: "my/API")!
    loader.load(from: url) { result = $0 }

    XCTAssertEqual(engine.requestedURL, url)
    XCTAssertEqual(result, .data("Hello world".data(using: .utf8)!))
}



```
