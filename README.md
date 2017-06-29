# An overview of mock tools in the Go language

This is an overview of the mock tools that I've tried.

### Disclaimer (or: _mockito is great_)
As a long-time Java developer, I've been spoiled by [Mockito](http://mockito.org/). It's only recently that I've made peace with the fact that an equivalent library is not possible in Go, because of the specifics of the language. However, many aspects such as the DSL and the assertions _should_ be possible in Go (and I miss them).

Here is what I found in my search for the Go-equivalent of Mockito.

### Why use a tool for creating mocks?
Mocks can be written by hand fairly easily, and I encourage everyone to do so. However, using mock tools should have a number of benefits:
- provide a uniform way to manipulate mocks accross a team
- make it easy to configure behavior on a test-by-test basis (implementin mocks manually usually means that we have a single mock implementation, which might lead to heavier coupling between tests if not careful)
- make some actions regarding mocks easier to understand; for example, a good DSL might make it more obvious when we are asserting that calls have been made, etc.
- encourage generating the mocks on the client side (based on local interfaces that wrap the external API), as opposed to providing them with a shared package

### What is mocked here
In this document, we have a `readHistory` function that calls `GetHistory` on something that implements `DataReader`:
```
type DataReader interface {
	GetHistory([]int64) ([]historyLine, error)
}
```

The issue is to get a mocked version `DataReader` that is easy to configure on a test-by-test basis.

## Mockery
[mockery](https://github.com/vektra/mockery) generates mocks in the `mocks` package, which is problematic when mocked functions take interfaces in parameter. This is because that interface (say, `mypackage.Data`) will also be mocked (as `mypackage.mocks.Data`); the mocked version will be in the signature of the functions (`myFunction(data mypackage.Data)` becomes `myFunction(data mypackage.mocks.Data)`), making them incompatible with the prod versions of the same function.

## Hel
[hel](https://github.com/nelsam/hel) generates mocks in current package. Unusually, the behavior of mocked functions is defined using chans. This is awkward when one forgets to specify all behaviors, as there will be a lock due to the channel waiting for the correct data.
Call assertions are done simply by passing the expected parameters to the channels.

Usage example:
```
hel --package ./...
```
Which creates a `helheim_test.go` file.

A sample test looks like that:
```
func Test_should_fail_when_the_storage_fails(t *testing.T) {
	var storage = newMockDataReader()
	storage.GetHistoryInput.Arg0 <- []int64{1}
	storage.GetHistoryOutput.Ret0 <- nil
	storage.GetHistoryOutput.Ret1 <- errors.New("")

	assert.Empty(t, readHistory(storage, "/?ad_id=1"))
}
```

## Moq
[moq](https://medium.com/@matryer/meet-moq-easily-mock-interfaces-in-go-476444187d10) generates mocks in the current package. Behavior is defined by creating special new functions.
Call assertions are done by manually checking parameter values. This is not ideal (the intention could be clearer to the person reading the code), but at least it is not very complicated to figure out.
Tolerating unexpected calls are not really a problem: just make the function return `nil` or some other appropriate default value. Conversely, if you want to be strict and fail when an unexpected call is made, just call `panic`.
The bigger issue is that import clauses are not generated. This means that functions that take or return imported classes cannot be mocked.

Usage example:
```
moq -out DataReader_test.go . DataReader
```
Which creates a `DataReader_test.go` file.

A sample test looks like that:
```
func Test_should_fail_when_the_storage_fails(t *testing.T) {
	storage := &DataReaderMock{
		GetHistoryFunc: func(in1 []int64) ([]historyLine, error) {
			if in1[0] == 1 && len(in1) == 1 {
				return nil, errors.New("something went wrong")
			}
			return nil, nil
		},
	}

	assert.Empty(t, readHistory(storage, "/?ad_id=1"))
}
```

## Pegomock
[pegomock](https://github.com/petergtz/pegomock) can only be run from a package that is exportable (package is not `main`), making it impossible to use it in a service or a script :-(
It also requires calling `pegomock.RegisterMockTestingT(t)`, which is rather ugly.

```
pegomock generate DataReader
```
(which gives `import "mypackage" is a program, not an importable package`, since this is indeed a service, not a shared package)

The DSL is really nice, although I couldn't try it in earnest (even reminiscent of Mockito):
```
display := NewMockDisplay()
sendStringToDisplay(display, "Hello World!")
display.VerifyWasCalledOnce().Show("Hello World!")
```

## Counterfeiter
[counterfeiter](https://github.com/maxbrunsfeld/counterfeiter) requires to be run from a package that is exportable (package is not `main`), making it impossible to use it in a service or a script :-(

Usage example:
```
counterfeiter  . DataReader
```
Which generates a file in `mypackagefakes/fake_data_reader.go`. This then fails when building on my project, as `fake_data_reader.go` will import the current package.

```
var display = new(mypackagefakes.FakeDisplay)
display.ShowStub = func(arg1 string) (int, error) {
	Expect(arg1).To(Equal("Hello World!"))
}

sendStringToDisplay(display, "Hello World!")
```
