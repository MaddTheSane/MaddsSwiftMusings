#Using Swift Enums with ErrorType and NSError

I've read some complaining on how a Swift `ErrorType` behaves when casted to an `NSError`. Some things about `ErrorType` `enum`s is that, when converted to an `NSError`, `NSError.code` is numbered in the order they're defined. I can see why this is needed: `enum`s can be of any type, not just integer types. Also, the error domain is defined as the module's name and the `enum`'s name.

But what if you want a different error domain name, or you have one you already use?

As an example, we'll take a look at [PlayerpRO](https://sourceforge.net/projects/playerpro/), a project I maintain. In the project there is an enum called `MADErr` (for all intents and purposes, `MADENUM` is just defined to `CF_ENUM`):

	typedef MADENUM(short, MADErr) {
		/// No error was encountered
		MADNoErr						= 0,
		/// There isn't enough memory to execute the command.
		MADNeedMemory 					= -1,
		/// An error occured when trying to reading a file.
		MADReadingErr					= -2,
		/// The file isn't compatible with PlayerPROCore.
		MADIncompatibleFile				= -3,
		/// A library function was called without an initialized library.
		MADLibraryNotInitialized		= -4,
		/// Bad paramaters were sent to a function.
		MADParametersErr				= -5,
		/// An unknown error occured.
		MADUnknownErr					= -6,
		/// An error occured when trying to initialize the sound system.
		MADSoundManagerErr				= -7,
		/// The plug-in doesn't implement the order specified.
		MADOrderNotImplemented			= -8,
		/// The file that a plug-in attempted to load was incompatible
		/// with said plug-in
		MADFileNotSupportedByThisPlug	= -9,
		/// PlayerPRO couldn't find the plug-in specified.
		MADCannotFindPlug				= -10,
		/// Attempted to use a music function that wasn't attached to a driver.
		MADMusicHasNoDriver				= -11,
		/// Attempted to use a driver function that requires the use of a loaded music file.
		MADDriverHasNoMusic				= -12,
		/// The sound system requested isn't available for the current architecture.
		MADSoundSystemUnavailable		= -13,
		/// An error occured when trying to write to a file
		MADWritingErr					= -14,
		/// The user cancelled an action. This shouldn't be seen by the user.
		MADUserCanceledErr				= -15
	};

If we only declare MADErr as such :

	extension MADErr: ErrorType {
		
	}

We don't get the result we want:

	do {
		throw MADErr.NeedMemory
	} catch let error as NSError {
		error.code //returns 1, not -1
		error.domain //returns "__C.MADErr"
	}

Problem is, in *PlayerPROKit* (the Objective-C bindings for `PlayerPROCore`, where `MADErr` is defined) , we have our own error domain, `PPMADErrorDomain`, which is used by the Objective-C code to display our own errors based off of `MADErr`. I have a function that wraps an `MADErr` in Cocoa's `NSError`, `PPCreateErrorFromMADErrorType`. Since Swift 2.0, support for error handling has made it easy to catch errors created by Objective-C methods. If we try to catch those errors as `MADErr`, it doesn't work. We need a solution.

###Private ErrorType Variables
It turns out that Swift's `ErrorType` protocol has a couple of hidden variables:

	protocol ErrorType {
	  var _domain: String { get }
	  var _code: Int { get }
	}

With this in mind, we can work some magic with our `MADErr` extension:

	extension MADErr: ErrorType {
		public var _domain: String {
			return PPMADErrorDomain
		}
		
		/// Use the value's error code
		public var _code: Int {
			return Int(rawValue)
		}
	}

As an added bonus, I've used 10.11's `+[NSError setUserInfoValueProviderForDomain:provider:]` so that if a thrown `MADErr` is converted to an `NSError`, the `userDictionary` is populated in the same way as if it was called by `PPCreateErrorFromMADErrorType`.

##Sources

* [Testing Swift's ErrorType: An Exploration](https://realm.io/news/testing-swift-error-type/)
